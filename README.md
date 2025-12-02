<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <title>手势控制 3D 粒子系统</title>
    <style>
        body { margin: 0; overflow: hidden; background-color: #050505; font-family: 'Segoe UI', sans-serif; }
        #canvas-container { width: 100vw; height: 100vh; position: absolute; top: 0; left: 0; z-index: 1; }
        
        /* 摄像头预览（可选隐藏或显示在角落） */
        #video-input {
            position: absolute; bottom: 20px; left: 20px; width: 160px; height: 120px;
            z-index: 2; border-radius: 10px; border: 2px solid rgba(255,255,255,0.3);
            transform: scaleX(-1); /* 镜像翻转 */
            opacity: 0.7;
        }

        /* 加载提示 */
        #loader {
            position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
            color: white; z-index: 10; font-size: 24px; pointer-events: none;
            transition: opacity 0.5s;
        }
        
        /* 全屏按钮 */
        #fullscreen-btn {
            position: absolute; top: 20px; left: 20px; z-index: 10;
            background: rgba(255, 255, 255, 0.1); border: 1px solid rgba(255, 255, 255, 0.2);
            color: white; padding: 8px 16px; border-radius: 20px; cursor: pointer;
            backdrop-filter: blur(5px); transition: 0.3s;
        }
        #fullscreen-btn:hover { background: rgba(255, 255, 255, 0.3); }
    </style>
</head>
<body>

    <div id="loader">正在初始化摄像头和 AI 模型...</div>
    <button id="fullscreen-btn">⛶ 全屏模式</button>
    <video id="video-input" playsinline></video>
    <div id="canvas-container"></div>

    <script type="importmap">
        {
            "imports": {
                "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
                "three/addons/": "https://unpkg.com/three@0.160.0/examples/jsm/"
            }
        }
    </script>

    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/control_utils/control_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/drawing_utils/drawing_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js" crossorigin="anonymous"></script>

    <script type="module">
        import * as THREE from 'three';
        import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
        import { GUI } from 'three/addons/libs/lil-gui.module.min.js';

        // --- 全局变量 ---
        let scene, camera, renderer, particles, geometry;
        let controls;
        let handLandmarks = null;
        let particleCount = 25000; // 粒子数量
        
        // 状态管理
        const state = {
            currentShape: 'heart', // 当前形状
            color: '#ff0055',      // 粒子颜色
            handInfluence: 0,      // 手势影响因子 (0=闭合, 1=张开)
            baseSize: 0.08,        // 粒子基础大小
            explosion: 0,          // 爆炸/扩散程度
            autoRotate: true
        };

        // 存储不同形状的目标位置数据
        const shapeGeometries = {};

        // --- 1. 初始化 Three.js 场景 ---
        function init() {
            const container = document.getElementById('canvas-container');

            scene = new THREE.Scene();
            // 稍微带点雾气增加深度感
            scene.fog = new THREE.FogExp2(0x050505, 0.002);

            camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
            camera.position.z = 30;

            renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
            container.appendChild(renderer.domElement);

            controls = new OrbitControls(camera, renderer.domElement);
            controls.enableDamping = true;
            controls.autoRotate = true;
            controls.autoRotateSpeed = 1.0;

            // 初始化粒子系统
            initParticles();

            // 初始化 UI
            initGUI();

            // 监听窗口大小变化
            window.addEventListener('resize', onWindowResize);
            
            // 全屏按钮
            document.getElementById('fullscreen-btn').addEventListener('click', toggleFullScreen);

            // 开始动画循环
            animate();
        }

        // --- 2. 粒子系统逻辑 ---
        function initParticles() {
            geometry = new THREE.BufferGeometry();
            
            const positions = new Float32Array(particleCount * 3);
            const initialPos = calculateHeart(particleCount); // 默认爱心

            // 设置初始位置
            for (let i = 0; i < particleCount * 3; i++) {
                positions[i] = initialPos[i];
            }

            geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));

            // 创建材质
            const material = new THREE.PointsMaterial({
                color: state.color,
                size: state.baseSize,
                sizeAttenuation: true,
                blending: THREE.AdditiveBlending,
                depthWrite: false,
                transparent: true,
                opacity: 0.8
            });

            // 创建贴图（用代码生成一个简单的发光点贴图，避免外部图片跨域问题）
            const sprite = createCircleTexture();
            material.map = sprite;

            particles = new THREE.Points(geometry, material);
            scene.add(particles);

            // --- 预计算所有形状 ---
            console.log("正在预计算形状...");
            shapeGeometries['heart'] = calculateHeart(particleCount);
            shapeGeometries['flower'] = calculateFlower(particleCount);
            shapeGeometries['saturn'] = calculateSaturn(particleCount);
            shapeGeometries['torus'] = calculateTorusKnot(particleCount); // 代替佛像的复杂结构
            shapeGeometries['fireworks'] = calculateSphere(particleCount, 10); // 烟花初始状态是个球，靠扩散炸开
        }

        // 辅助：创建一个圆形渐变贴图
        function createCircleTexture() {
            const canvas = document.createElement('canvas');
            canvas.width = 32; canvas.height = 32;
            const context = canvas.getContext('2d');
            const gradient = context.createRadialGradient(16, 16, 0, 16, 16, 16);
            gradient.addColorStop(0, 'rgba(255,255,255,1)');
            gradient.addColorStop(0.2, 'rgba(255,255,255,0.8)');
            gradient.addColorStop(0.5, 'rgba(255,255,255,0.2)');
            gradient.addColorStop(1, 'rgba(0,0,0,0)');
            context.fillStyle = gradient;
            context.fillRect(0, 0, 32, 32);
            const texture = new THREE.CanvasTexture(canvas);
            return texture;
        }

        // --- 3. 形状数学生成算法 ---

        // 爱心
        function calculateHeart(count) {
            const arr = new Float32Array(count * 3);
            for (let i = 0; i < count; i++) {
                let t = Math.random() * Math.PI * 2;
                let u = Math.random() * Math.PI * 2; 
                // 稍微随机化分布以填充体积
                let x = 16 * Math.pow(Math.sin(t), 3);
                let y = 13 * Math.cos(t) - 5 * Math.cos(2*t) - 2 * Math.cos(3*t) - Math.cos(4*t);
                let z = (Math.random() - 0.5) * 5; // 厚度
                
                // 缩放
                const scale = 0.5;
                arr[i*3] = x * scale;
                arr[i*3+1] = y * scale;
                arr[i*3+2] = z * scale;
            }
            return arr;
        }

        // 菲涅耳花朵 (Phyllotaxis)
        function calculateFlower(count) {
            const arr = new Float32Array(count * 3);
            const goldenAngle = Math.PI * (3 - Math.sqrt(5));
            for (let i = 0; i < count; i++) {
                const r = 0.3 * Math.sqrt(i);
                const theta = i * goldenAngle;
                
                // 花瓣形状变换
                const x = r * Math.cos(theta);
                const y = r * Math.sin(theta);
                const z = Math.sin(r * 0.5) * 5 - 2; 

                arr[i*3] = x;
                arr[i*3+1] = y; // 旋转一下方向
                arr[i*3+2] = z;
            }
            return arr;
        }

        // 土星 (球体 + 环)
        function calculateSaturn(count) {
            const arr = new Float32Array(count * 3);
            const ringStart = count * 0.7; // 70% 粒子做球体，30% 做环
            
            for (let i = 0; i < count; i++) {
                if (i < ringStart) {
                    // 球体
                    const r = 6;
                    const theta = Math.random() * Math.PI * 2;
                    const phi = Math.acos(2 * Math.random() - 1);
                    arr[i*3] = r * Math.sin(phi) * Math.cos(theta);
                    arr[i*3+1] = r * Math.sin(phi) * Math.sin(theta);
                    arr[i*3+2] = r * Math.cos(phi);
                } else {
                    // 环
                    const r = 9 + Math.random() * 4; // 半径 9-13
                    const theta = Math.random() * Math.PI * 2;
                    arr[i*3] = r * Math.cos(theta);
                    arr[i*3+1] = (Math.random() - 0.5) * 0.5; // 环很薄
                    arr[i*3+2] = r * Math.sin(theta);
                }
            }
            return arr;
        }

        // 环形扭结 (复杂结构，替代佛像)
        function calculateTorusKnot(count) {
            const arr = new Float32Array(count * 3);
            for(let i = 0; i < count; i++) {
                const t = (i / count) * Math.PI * 2 * 10; // 缠绕圈数
                const p = 2, q = 3; // p, q 决定扭结形状
                const r = 4 + Math.cos(q * t) * 1.5; // 管的粗细
                
                // 增加一点随机散布让它看起来像粒子云而不是线条
                const spread = (Math.random() - 0.5) * 1.5;

                const x = r * Math.cos(p * t) + spread;
                const y = r * Math.sin(p * t) + spread;
                const z = 3 * Math.sin(q * t) + spread; // 拉伸高度

                arr[i*3] = x;
                arr[i*3+1] = y;
                arr[i*3+2] = z;
            }
            return arr;
        }

        // 球体 (用于烟花初始态)
        function calculateSphere(count, radius) {
            const arr = new Float32Array(count * 3);
            for (let i = 0; i < count; i++) {
                const r = radius * Math.cbrt(Math.random()); // 均匀分布
                const theta = Math.random() * Math.PI * 2;
                const phi = Math.acos(2 * Math.random() - 1);
                arr[i*3] = r * Math.sin(phi) * Math.cos(theta);
                arr[i*3+1] = r * Math.sin(phi) * Math.sin(theta);
                arr[i*3+2] = r * Math.cos(phi);
            }
            return arr;
        }


        // --- 4. 动画与 Morphing ---
        function animate() {
            requestAnimationFrame(animate);

            controls.autoRotate = state.autoRotate;
            controls.update();

            // 核心逻辑：粒子插值移动
            const positions = particles.geometry.attributes.position.array;
            const target = shapeGeometries[state.currentShape];

            // 交互影响：根据手势计算额外的扩散因子
            // handInfluence: 0 (正常) -> 1 (手张开/最大距离)
            // 我们让手张开时，粒子放大且稍微扩散；手合拢时，粒子紧缩
            
            // 基础变换速度
            const speed = 0.05; 
            
            // 手势控制的缩放倍率 (1 到 3倍)
            let scaleFactor = 1 + (state.handInfluence * 1.5); 
            // 烟花特殊处理：如果选中烟花，手张开就像爆炸
            if (state.currentShape === 'fireworks') {
                scaleFactor = 1 + (state.handInfluence * 4.0);
            }

            for (let i = 0; i < particleCount; i++) {
                const ix = i * 3;
                const iy = i * 3 + 1;
                const iz = i * 3 + 2;

                // 目标位置 (带上手势缩放)
                const tx = target[ix] * scaleFactor;
                const ty = target[iy] * scaleFactor;
                const tz = target[iz] * scaleFactor;

                // 简单的线性插值 (Lerp)
                positions[ix] += (tx - positions[ix]) * speed;
                positions[iy] += (ty - positions[iy]) * speed;
                positions[iz] += (tz - positions[iz]) * speed;

                // 如果是烟花模式，增加颤动效果
                if (state.currentShape === 'fireworks' && state.handInfluence > 0.5) {
                    positions[ix] += (Math.random() - 0.5) * 0.5;
                    positions[iy] += (Math.random() - 0.5) * 0.5;
                    positions[iz] += (Math.random() - 0.5) * 0.5;
                }
            }

            particles.geometry.attributes.position.needsUpdate = true;
            
            // 实时更新颜色
            particles.material.color.set(state.color);
            // 实时更新粒子大小 (手张开粒子变大一点)
            particles.material.size = state.baseSize * (1 + state.handInfluence * 0.5);

            renderer.render(scene, camera);
        }


        // --- 5. MediaPipe Hands 集成 ---
        function setupMediaPipe() {
            const videoElement = document.getElementById('video-input');
            
            const hands = new Hands({locateFile: (file) => {
                return `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`;
            }});

            hands.setOptions({
                maxNumHands: 1,
                modelComplexity: 1,
                minDetectionConfidence: 0.5,
                minTrackingConfidence: 0.5
            });

            hands.onResults(onHandsResults);

            const cameraUtils = new Camera(videoElement, {
                onFrame: async () => {
                    await hands.send({image: videoElement});
                },
                width: 320,
                height: 240
            });
            
            cameraUtils.start()
                .then(() => {
                    document.getElementById('loader').style.display = 'none';
                    console.log("摄像头启动成功");
                })
                .catch(err => {
                    document.getElementById('loader').innerText = "摄像头访问失败，请检查权限。";
                    console.error(err);
                });
        }

        function onHandsResults(results) {
            if (results.multiHandLandmarks && results.multiHandLandmarks.length > 0) {
                const landmarks = results.multiHandLandmarks[0];
                handLandmarks = landmarks;

                // 计算 大拇指尖(4) 和 食指尖(8) 的距离
                const thumbTip = landmarks[4];
                const indexTip = landmarks[8];
                
                // 简单的欧几里得距离 (只考虑 x, y，因为 z 深度在普通摄像头下不够准)
                const distance = Math.sqrt(
                    Math.pow(thumbTip.x - indexTip.x, 2) + 
                    Math.pow(thumbTip.y - indexTip.y, 2)
                );

                // 归一化距离：通常手捏合时约为0.02，张开最大约为0.2-0.3（取决于摄像头距离）
                // 我们映射到 0 ~ 1 之间
                let normalized = (distance - 0.02) / 0.2; 
                normalized = Math.max(0, Math.min(1, normalized)); // Clamp

                // 平滑处理 (简单的缓动)
                state.handInfluence += (normalized - state.handInfluence) * 0.1;
            } else {
                // 没有检测到手时，慢慢归零
                state.handInfluence += (0 - state.handInfluence) * 0.05;
            }
        }


        // --- 6. UI 面板 ---
        function initGUI() {
            const gui = new GUI({ title: '控制面板' });
            
            // 形状选择
            gui.add(state, 'currentShape', {
                '爱心 Heart': 'heart',
                '花朵 Flower': 'flower',
                '土星 Saturn': 'saturn',
                '环形结 (Complex)': 'torus',
                '烟花 Fireworks': 'fireworks'
            }).name('模型选择 Shape');

            // 颜色
            gui.addColor(state, 'color').name('粒子颜色 Color');
            
            // 大小
            gui.add(state, 'baseSize', 0.01, 0.3).name('粒子大小 Size');

            // 自动旋转开关
            gui.add(state, 'autoRotate').name('自动旋转 Rotate');

            // 调试显示手势值
            gui.add(state, 'handInfluence', 0, 1).name('手势强度 (只读)').listen();

            // 样式调整
            gui.domElement.style.top = '20px';
            gui.domElement.style.right = '20px';
        }

        // --- 辅助功能 ---
        function onWindowResize() {
            camera.aspect = window.innerWidth / window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        }

        function toggleFullScreen() {
            if (!document.fullscreenElement) {
                document.documentElement.requestFullscreen();
            } else {
                if (document.exitFullscreen) {
                    document.exitFullscreen();
                }
            }
        }

        // 启动
        init();
        setupMediaPipe();

    </script>
</body>
</html>
