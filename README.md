<!DOCTYPE html>
<html lang="pt">
<head>
    <meta charset="UTF-8">
    <title>Three.js - Navegação com Cubo Interativo</title>

    <!-- Importmap para Three.js e addons -->
    <script type="importmap">
        {
            "imports": {
                "three": "https://cdn.jsdelivr.net/npm/three@0.133.0/build/three.module.js",
                "three/addons/": "https://cdn.jsdelivr.net/npm/three@0.133.0/examples/jsm/"
            }
        }
    </script>

    <script type="module">
        import * as THREE from 'three';
        import { OBJLoader } from 'three/addons/loaders/OBJLoader.js';
        import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

        let camera, scene, renderer, controls, object, axesGroup, navCube, raycaster, mouse, targetPosition;

        function init() {
            // Configurar câmera
            camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 2000);
            camera.position.set(-1000, 300, 0); // Posição inicial
            camera.lookAt(new THREE.Vector3(0, 0, 0)); // Olhar para o centro da cena

            // Criar a cena
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0xffffff); // Fundo branco

            // Adicionar eixos coloridos e pontilhados
            addDashedAxes();

            // Adicionar luz ambiente suave
            const ambientLight = new THREE.AmbientLight(0xffffff, 0.05);
            scene.add(ambientLight);

            // Adicionar luz direcional para destaque
            const directionalLight = new THREE.DirectionalLight(0xffffff, 1);
            directionalLight.position.set(2, 2, 2).normalize();
            scene.add(directionalLight);

            // Configurar renderizador com antialiasing e profundidade logarítmica
            renderer = new THREE.WebGLRenderer({ antialias: true, logarithmicDepthBuffer: true });
            renderer.setPixelRatio(window.devicePixelRatio);
            renderer.setSize(window.innerWidth, window.innerHeight);
            document.body.appendChild(renderer.domElement);

            // Configurar os controles de órbita
            controls = new OrbitControls(camera, renderer.domElement);
            controls.enableDamping = true;
            controls.dampingFactor = 0.3;
            controls.minDistance = 1;
            controls.maxDistance = 100;
            controls.zoomSpeed = 1.0;

            // Adicionar o cubo de navegação
            addNavigationCube();

            // Configurar raycaster e mouse para detecção de cliques
            raycaster = new THREE.Raycaster();
            mouse = new THREE.Vector2();

            // Evento para detectar cliques
            window.addEventListener('click', onCubeClick);

            // Carregar o modelo OBJ e aplicar material transparente com rotação fixa
            loadModel();

            // Ajustar renderização ao redimensionar a janela
            window.addEventListener('resize', onWindowResize);

            // Inicializar a posição alvo para animação suave
            targetPosition = new THREE.Vector3();
        }

        function addNavigationCube() {
            // Criar o cubo de navegação com faces rotuladas
            const materials = [
                new THREE.MeshBasicMaterial({ color: 0xcccccc, side: THREE.DoubleSide, map: createLabelTexture("DIREITA") }),  // Face 0 - Direita
                new THREE.MeshBasicMaterial({ color: 0xcccccc, side: THREE.DoubleSide, map: createLabelTexture("ESQUERDA") }), // Face 1 - Esquerda
                new THREE.MeshBasicMaterial({ color: 0xcccccc, side: THREE.DoubleSide, map: createLabelTexture("SUPERIOR") }), // Face 2 - Superior
                new THREE.MeshBasicMaterial({ color: 0xcccccc, side: THREE.DoubleSide, map: createLabelTexture("INFERIOR") }), // Face 3 - Inferior
                new THREE.MeshBasicMaterial({ color: 0xcccccc, side: THREE.DoubleSide, map: createLabelTexture("FRONTAL") }),  // Face 4 - Frontal
                new THREE.MeshBasicMaterial({ color: 0xcccccc, side: THREE.DoubleSide, map: createLabelTexture("POSTERIOR") }) // Face 5 - Posterior
            ];

            navCube = new THREE.Mesh(new THREE.BoxGeometry(10, 10, 10), materials);
            navCube.position.set(50, 65, 80); // Posição no canto superior direito
            scene.add(navCube);
        }

        function createLabelTexture(text) {
            const canvas = document.createElement('canvas');
            const size = 170;
            canvas.width = size;
            canvas.height = size;
            const context = canvas.getContext('2d');
            context.fillStyle = '#f2f2f2';
            context.fillRect(0, 0, size, size);
            context.fillStyle = '#000000';
            context.font = '24px Arial';
            context.textAlign = 'center';
            context.textBaseline = 'middle';
            context.fillText(text, size / 2, size / 2);
            return new THREE.CanvasTexture(canvas);
        }

        function onCubeClick(event) {
            // Converter posição do mouse para coordenadas normalizadas
            mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
            mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

            // Atualizar raycaster com a posição do mouse
            raycaster.setFromCamera(mouse, camera);

            // Verificar interseção com o cubo de navegação
            const intersects = raycaster.intersectObject(navCube, true);
            if (intersects.length > 0) {
                const faceIndex = intersects[0].faceIndex;

                // Definir a posição alvo para a câmera com base na face clicada
                switch (faceIndex) {
                    case 0: // Direita
                        targetPosition.set(100, 0, 0);
                        break;
                    case 1: // Esquerda
                        targetPosition.set(-100, 0, 0);
                        break;
                    case 2: // Superior
                        targetPosition.set(0, 100, 0);
                        break;
                    case 3: // Inferior
                        targetPosition.set(0, -100, 0);
                        break;
                    case 4: // Frontal
                        targetPosition.set(0, 0, 100);
                        break;
                    case 5: // Posterior
                        targetPosition.set(0, 0, -100);
                        break;
                }
                animateCameraToTarget();
            }
        }

        function animateCameraToTarget() {
            // Animação suave para mover a câmera até a posição alvo
            const duration = 1000; // Duração em milissegundos
            const startPosition = camera.position.clone();
            const startTime = performance.now();

            function animate() {
                const elapsed = performance.now() - startTime;
                const t = Math.min(elapsed / duration, 1); // Limitar entre 0 e 1
                camera.position.lerpVectors(startPosition, targetPosition, t);
                camera.lookAt(object.position);
                controls.update();

                if (t < 1) {
                    requestAnimationFrame(animate);
                }
            }
            animate();
        }

        function loadModel() {
            const loader = new OBJLoader();
            loader.load(
                './models/ROBO.obj',
                function (obj) {
                    obj.traverse(function (child) {
                        if (child.isMesh) {
                            child.material = new THREE.MeshStandardMaterial({
                                color: new THREE.Color(0x87CEFA),
                                transparent: true,
                                opacity: 0.2,
                                side: THREE.DoubleSide,
                                depthWrite: false
                            });
                        }
                    });

                    obj.scale.set(1.8, 1.8, 1.8);
                    obj.rotation.set(0, -0.6, 0);
                    scene.add(obj);
                    object = obj;
                    centerCameraOnObject(obj);
                    animate();
                },
                undefined,
                function (error) {
                    console.error('Erro ao carregar o modelo OBJ:', error);
                }
            );
        }

        function centerCameraOnObject(obj) {
            const box = new THREE.Box3().setFromObject(obj);
            const center = box.getCenter(new THREE.Vector3());
            const size = box.getSize(new THREE.Vector3());
            const maxDim = Math.max(size.x, size.y, size.z);
            const fov = camera.fov * (Math.PI / 180);
            const cameraZ = Math.abs(maxDim / 2 / Math.tan(fov / 2));

            camera.position.z = cameraZ * 1.5;
            camera.lookAt(center);
            controls.target.copy(center);
            controls.update();
        }

        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        }

        function animate() {
            requestAnimationFrame(animate);
            controls.update();
            renderer.render(scene, camera);
        }

        function addDashedAxes() {
            axesGroup = new THREE.Group();
            const colors = { x: 0xff0000, y: 0x00ff00, z: 0x0000ff };

            ['x', 'y', 'z'].forEach((axis, i) => {
                const geometrySolid = new THREE.BufferGeometry().setFromPoints([
                    new THREE.Vector3(0, 0, 0),
                    new THREE.Vector3(i === 0 ? 500 : 0, i === 1 ? 500 : 0, i === 2 ? 500 : 0)
                ]);
                const materialSolid = new THREE.LineBasicMaterial({ color: colors[axis] });
                const lineSolid = new THREE.Line(geometrySolid, materialSolid);
                axesGroup.add(lineSolid);

                const geometryDashed = new THREE.BufferGeometry().setFromPoints([
                    new THREE.Vector3(0, 0, 0),
                    new THREE.Vector3(i === 0 ? -500 : 0, i === 1 ? -500 : 0, i === 2 ? -500 : 0)
                ]);
                const materialDashed = new THREE.LineDashedMaterial({
                    color: colors[axis],
                    dashSize: 0.5,
                    gapSize: 0.5
                });
                const lineDashed = new THREE.Line(geometryDashed, materialDashed);
                lineDashed.computeLineDistances();
                axesGroup.add(lineDashed);
            });

            axesGroup.rotation.set(Math.PI / -2, 0, 2.542);
            scene.add(axesGroup);
        }

        init();
    </script>
</head>
<body>
</body>
</html>
