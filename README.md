[index.html](https://github.com/user-attachments/files/24082463/index.html)
<!DOCTYPE html>
<html lang="zh">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>千里江山图 | 交互粒子卷轴</title>
    <style>
        body { margin: 0; overflow: hidden; background-color: #05101a; font-family: "Microsoft YaHei", sans-serif; }
        
        /* 错误诊断层 */
        #debug-console {
            position: fixed; top: 0; left: 0; width: 100%; height: 100%;
            background: rgba(0,0,0,0.95); color: #0f0; padding: 20px;
            display: none; z-index: 9999; white-space: pre-wrap; font-family: monospace;
        }

        /* 顶部 UI */
        #ui-container {
            position: absolute; top: 20px; left: 50%; transform: translateX(-50%);
            z-index: 10; display: flex; gap: 12px; flex-wrap: wrap; justify-content: center;
        }
        
        .mode-btn {
            background: rgba(18, 55, 64, 0.4); 
            border: 1px solid rgba(64, 224, 208, 0.3);
            color: #b0d0d3; padding: 8px 18px; border-radius: 4px; 
            cursor: pointer; backdrop-filter: blur(4px); transition: 0.4s;
            font-size: 14px; letter-spacing: 1px;
        }
        .mode-btn:hover, .mode-btn.active { 
            background: rgba(64, 224, 208, 0.2); 
            border-color: #40e0d0; color: #fff; box-shadow: 0 0 15px rgba(64,224,208,0.2);
        }

        /* 加载层 */
        #loader {
            position: fixed; top: 0; left: 0; width: 100%; height: 100%;
            background: #05101a; z-index: 500; display: flex; flex-direction: column;
            justify-content: center; align-items: center; color: #5a7d8a;
            transition: opacity 0.8s;
        }
        .spinner {
            width: 50px; height: 50px; border: 2px solid #1a3a4a;
            border-top: 2px solid #40e0d0; border-radius: 50%;
            animation: spin 1.5s linear infinite; margin-bottom: 20px;
        }
        @keyframes spin { 0% { transform: rotate(0deg); } 100% { transform: rotate(360deg); } }

        /* 全屏按钮 */
        #fs-btn {
            position: absolute; bottom: 20px; right: 20px; z-index: 10;
            background: none; border: 1px solid rgba(255,255,255,0.2);
            color: rgba(255,255,255,0.6); padding: 6px 10px; cursor: pointer;
        }
    </style>

    <!-- 核心库 -->
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/control_utils/control_utils.js" crossorigin="anonymous"></script>
    <script src="https://cdn.jsdelivr.net/npm/@mediapipe/hands/hands.js" crossorigin="anonymous"></script>
    
    <script>
        // 错误捕获
        window.onerror = function(msg) {
            const d = document.getElementById('debug-console');
            d.style.display = 'block';
            d.innerHTML += `[错误] ${msg}\n请确保使用 HTTPS 或本地服务器打开。`;
        }
    </script>
</head>
<body>

    <div id="debug-console"><h3>🛑 运行诊断日志</h3></div>

    <div id="loader">
        <div class="spinner"></div>
        <div>笔墨晕染中...</div>
    </div>

    <div id="ui-container">
        <button class="mode-btn active" onclick="setMode('landscape')">⛰️ 千里江山</button>
        <button class="mode-btn" onclick="setMode('text', '龙年大吉')">🐲 龙年大吉</button>
        <button class="mode-btn" onclick="setMode('text', '新年快乐')">🧧 新年快乐</button>
        <button class="mode-btn" onclick="setMode('text', '万事如意')">✨ 万事如意</button>
    </div>

    <video id="video-input" style="display:none" playsinline></video>
    <div id="canvas-container"></div>
    <button id="fs-btn">⛶ 全屏赏画</button>

    <script type="importmap">
        {
            "imports": {
                "three": "https://unpkg.com/three@0.160.0/build/three.module.js",
                "three/addons/": "https://unpkg.com/three@0.160.0/examples/jsm/",
                "lil-gui": "https://unpkg.com/lil-gui@0.19.1/dist/lil-gui.esm.min.js"
            }
        }
    </script>

    <script type="module">
        import * as THREE from 'three';
        import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
        import GUI from 'lil-gui';

        // --- 1. 配置与配色 (青绿山水) ---
        const config = {
            count: 15000,        // 粒子总数
            size: 0.18,          // 粒子大小
            interactStrength: 0, // 手势强度 (0=散, 1=聚)
            mode: 'landscape',   // 当前模式
            flowSpeed: 0.2       // 浮动速度
        };

        // 千里江山图配色表
        const PALETTE = {
            shiQing: new THREE.Color('#104E8B'), // 石青 (深蓝)
            shiLv:   new THREE.Color('#2E8B57'), // 石绿 (翠绿)
            zheShi:  new THREE.Color('#CD853F'), // 赭石 (山脚/土地)
            mist:    new THREE.Color('#AFEEEE'), // 烟波 (浅青)
            ink:     new THREE.Color('#000000')  // 水墨
        };

        // --- 2. 场景初始化 ---
        if (window.location.protocol === 'file:') throw new Error("请勿直接双击打开");

        const container = document.getElementById('canvas-container');
        const scene = new THREE.Scene();
        // 背景色：深邃的绢本底色
        scene.background = new THREE.Color('#05101a'); 
        scene.fog = new THREE.FogExp2(0x05101a, 0.015); // 雾气增强景深

        const camera = new THREE.PerspectiveCamera(60, window.innerWidth/window.innerHeight, 0.1, 200);
        camera.position.set(0, 0, 25);

        const renderer = new THREE.WebGLRenderer({ antialias: true });
        renderer.setSize(window.innerWidth, window.innerHeight);
        renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2));
        container.appendChild(renderer.domElement);

        const controls = new OrbitControls(camera, renderer.domElement);
        controls.enableDamping = true;
        controls.autoRotate = true;
        controls.autoRotateSpeed = 0.5;
        controls.maxPolarAngle = Math.PI / 1.5; // 限制视角不能钻到地底

        // --- 3. 粒子画笔纹理 ---
        // 制作边缘柔和的圆形纹理，模拟水墨点
        const brushTexture = (() => {
            const canvas = document.createElement('canvas');
            canvas.width = 32; canvas.height = 32;
            const ctx = canvas.getContext('2d');
            const grad = ctx.createRadialGradient(16,16,0,16,16,16);
            grad.addColorStop(0, 'rgba(255,255,255,1)');
            grad.addColorStop(0.5, 'rgba(255,255,255,0.3)');
            grad.addColorStop(1, 'rgba(255,255,255,0)');
            ctx.fillStyle = grad;
            ctx.fillRect(0, 0, 32, 32);
            return new THREE.Texture(canvas);
        })();
        brushTexture.needsUpdate = true;

        // --- 4. 粒子系统核心 ---
        let particles;
        const geometry = new THREE.BufferGeometry();
        
        // 数据数组
        const positions = new Float32Array(config.count * 3); // 当前位置
        const targetPos = new Float32Array(config.count * 3); // 目标形态 (山或字)
        const scatterPos = new Float32Array(config.count * 3); // 烟雾形态 (散开)
        const colors = new Float32Array(config.count * 3);
        const layers = new Float32Array(config.count); // 记录属于哪一层 (0-3)
        const randoms = new Float32Array(config.count); // 随机因子用于呼吸动效

        // 初始化：生成烟雾散点 + 分配层级
        for(let i=0; i<config.count; i++) {
            const i3 = i*3;
            
            // 散开形态：像云雾一样弥漫整个空间
            scatterPos[i3] = (Math.random()-0.5) * 50;
            scatterPos[i3+1] = (Math.random()-0.5) * 30;
            scatterPos[i3+2] = (Math.random()-0.5) * 20;

            // 初始位置 = 散开
            positions[i3] = scatterPos[i3];
            positions[i3+1] = scatterPos[i3+1];
            positions[i3+2] = scatterPos[i3+2];

            randoms[i] = Math.random();
            // 随机分配层级 (0:远山, 1:中山, 2:近丘, 3:水面/前景)
            layers[i] = Math.floor(Math.random() * 4);
        }

        geometry.setAttribute('position', new THREE.BufferAttribute(positions, 3));
        geometry.setAttribute('color', new THREE.BufferAttribute(colors, 3));

        const material = new THREE.PointsMaterial({
            size: config.size,
            map: brushTexture,
            vertexColors: true,
            blending: THREE.AdditiveBlending,
            depthWrite: false,
            transparent: true,
            opacity: 0.85
        });

        particles = new THREE.Points(geometry, material);
        scene.add(particles);

        // --- 5. 形态生成算法 ---

        /**
         * 算法 A：生成层峦叠穿的山脉 (Math.sin 叠加)
         */
        function generateLandscape() {
            const width = 40;
            
            for(let i=0; i<config.count; i++) {
                const i3 = i*3;
                const layer = layers[i];
                const rnd = randoms[i];

                // X轴：均匀分布
                let x = (rnd - 0.5) * width;
                let y = 0;
                let z = 0;
                let col = new THREE.Color();

                // 根据层级使用不同的数学公式生成山形
                if (layer === 0) { 
                    // 远山 (石青 - 高且平缓)
                    z = -8 + (rnd * 2);
                    y = Math.sin(x * 0.15) * 4 + Math.cos(x * 0.4) * 1 + 2;
                    col.copy(PALETTE.shiQing);
                } else if (layer === 1) {
                    // 中山 (石绿 - 陡峭)
                    z = -2 + (rnd * 3);
                    y = Math.sin(x * 0.3) * 3 + Math.abs(Math.cos(x * 0.8)) * 2 - 1;
                    // 给山顶加点青色，山脚绿色
                    col.copy(PALETTE.shiLv).lerp(PALETTE.shiQing, y/6); 
                } else if (layer === 2) {
                    // 近丘 (赭石 - 低矮)
                    z = 4 + (rnd * 2);
                    y = Math.sin(x * 0.5 + 2) * 1.5 + Math.cos(x * 1.5) * 0.5 - 4;
                    col.copy(PALETTE.zheShi);
                } else {
                    // 水面/烟波 (散乱)
                    z = 8 + (rnd * 4);
                    y = -6 + (Math.sin(x * 0.2 + z) * 0.5);
                    col.copy(PALETTE.mist);
                }

                // 写入目标位置
                targetPos[i3] = x;
                targetPos[i3+1] = y;
                targetPos[i3+2] = z;

                // 颜色稍微随机化，模拟笔触浓淡
                col.r += (Math.random()-0.5)*0.1;
                col.g += (Math.random()-0.5)*0.1;
                col.b += (Math.random()-0.5)*0.1;
                
                colors[i3] = col.r; colors[i3+1] = col.g; colors[i3+2] = col.b;
            }
            geometry.attributes.color.needsUpdate = true;
        }

        /**
         * 算法 B：生成文字 (Canvas 采样)
         */
        function generateText(text) {
            const size = 1024;
            const cvs = document.createElement('canvas');
            cvs.width = size; cvs.height = size;
            const ctx = cvs.getContext('2d');
            
            ctx.fillStyle = '#000'; ctx.fillRect(0,0,size,size);
            ctx.fillStyle = '#fff'; 
            ctx.font = 'bold 300px "Microsoft YaHei", "Kaiti SC"'; // 使用楷体更有韵味
            ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
            ctx.fillText(text, size/2, size/2);

            const data = ctx.getImageData(0,0,size,size).data;
            const validPixels = [];

            // 稀疏采样
            for(let y=0; y<size; y+=5) {
                for(let x=0; x<size; x+=5) {
                    if(data[(y*size+x)*4] > 100) {
                        validPixels.push({
                            x: (x/size - 0.5) * 25,
                            y: -(y/size - 0.5) * 25,
                            z: 0
                        });
                    }
                }
            }

            // 分配粒子到文字点
            for(let i=0; i<config.count; i++) {
                const i3 = i*3;
                if (validPixels.length > 0) {
                    const p = validPixels[i % validPixels.length];
                    // 加一点厚度 Z
                    targetPos[i3] = p.x;
                    targetPos[i3+1] = p.y;
                    targetPos[i3+2] = (Math.random()-0.5) * 2;
                } else {
                    // 文字点不够时，剩下的粒子变成背景星光
                    targetPos[i3] = (Math.random()-0.5)*50;
                    targetPos[i3+1] = (Math.random()-0.5)*50;
                    targetPos[i3+2] = (Math.random()-0.5)*20;
                }

                // 文字颜色：金光闪闪
                const col = i % 5 === 0 ? PALETTE.mist : PALETTE.zheShi; // 混一点金色
                colors[i3] = col.r; colors[i3+1] = col.g; colors[i3+2] = col.b;
            }
            geometry.attributes.color.needsUpdate = true;
        }

        // 切换模式接口
        window.setMode = (mode, text) => {
            config.mode = mode;
            // 更新按钮样式
            document.querySelectorAll('.mode-btn').forEach(btn => btn.classList.remove('active'));
            event.target.classList.add('active');

            if(mode === 'landscape') {
                generateLandscape();
            } else {
                generateText(text);
            }
        };

        // 默认启动
        generateLandscape();


        // --- 6. 手势控制逻辑 ---
        const video = document.getElementById('video-input');
        const hands = new window.Hands({locateFile: (file) => `https://cdn.jsdelivr.net/npm/@mediapipe/hands/${file}`});
        
        hands.setOptions({ maxNumHands: 1, minDetectionConfidence: 0.5 });
        
        hands.onResults(results => {
            document.getElementById('loader').style.opacity = '0';
            setTimeout(()=>document.getElementById('loader').style.display='none', 800);

            if(results.multiHandLandmarks.length > 0) {
                const lm = results.multiHandLandmarks[0];
                const wrist = lm[0];
                
                // 计算五指张开程度
                let totalDist = 0;
                [4,8,12,16,20].forEach(tip => {
                    totalDist += Math.sqrt(Math.pow(lm[tip].x - wrist.x, 2) + Math.pow(lm[tip].y - wrist.y, 2));
                });
                const avgDist = totalDist / 5;

                // 映射逻辑：
                // 距离 > 0.3 (张手) -> 强度 1 (聚合成画)
                // 距离 < 0.15 (握拳) -> 强度 0 (散开成烟)
                const targetStrength = Math.min(Math.max((avgDist - 0.15) * 5, 0), 1);
                
                // 平滑过渡
                config.interactStrength += (targetStrength - config.interactStrength) * 0.1;
            } else {
                // 没检测到手，默认慢慢聚合成画 (美观考虑)
                config.interactStrength += (1.0 - config.interactStrength) * 0.05;
            }
        });

        const cam = new window.Camera(video, {
            onFrame: async()=>{ await hands.send({image:video}); },
            width:640, height:480
        });
        
        // 尝试启动摄像头，失败也不影响观看
        cam.start().catch(e => {
            console.log("摄像头未启动，进入自动演示模式");
            config.interactStrength = 1; // 默认展示画作
        });


        // --- 7. 动画渲染循环 ---
        const clock = new THREE.Clock();

        function animate() {
            requestAnimationFrame(animate);
            const time = clock.getElapsedTime();
            controls.update();

            // 粒子运动逻辑
            // strength 1 = 目标形态 (画/字)
            // strength 0 = 散开形态 (烟)
            const strength = config.interactStrength;

            for(let i=0; i<config.count; i++) {
                const i3 = i*3;
                
                // 1. 获取目标位置 (Target) 和 散开位置 (Scatter)
                const tx = targetPos[i3];
                const ty = targetPos[i3+1];
                const tz = targetPos[i3+2];

                const sx = scatterPos[i3];
                const sy = scatterPos[i3+1];
                const sz = scatterPos[i3+2];

                // 2. 根据手势强度混合位置 (Lerp)
                // 这里的混合点是 "理想位置"
                let ix = sx + (tx - sx) * strength;
                let iy = sy + (ty - sy) * strength;
                let iz = sz + (tz - sz) * strength;

                // 3. 添加 "气韵生动" 的呼吸噪点 (Perlin-ish Noise)
                // 当 strength=1 (成画) 时，只有微风吹拂的效果
                // 当 strength=0 (散开) 时，扰动更大
                const noiseAmp = strength > 0.8 ? 0.1 : 0.5;
                const rnd = randoms[i];
                
                const waveX = Math.sin(time * 0.5 + rnd * 10) * noiseAmp;
                const waveY = Math.cos(time * 0.3 + rnd * 20) * noiseAmp;
                
                // 4. 更新粒子实际位置 (缓动跟随)
                // 让粒子像墨水在水中晕开一样，有个延迟感
                positions[i3]   += (ix + waveX - positions[i3]) * 0.08;
                positions[i3+1] += (iy + waveY - positions[i3]) * 0.08;
                positions[i3+2] += (iz - positions[i3+2]) * 0.08;
            }

            geometry.attributes.position.needsUpdate = true;
            renderer.render(scene, camera);
        }

        animate();

        // 窗口适配
        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth/window.innerHeight;
            camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });

        // 全屏逻辑
        document.getElementById('fs-btn').onclick = () => {
            if(!document.fullscreenElement) document.documentElement.requestFullscreen();
            else document.exitFullscreen();
        };

        // GUI 调试面板 (可选颜色调整)
        const gui = new GUI({ title: '画笔设置' });
        gui.add(config, 'interactStrength', 0, 1).name('手势开合模拟').listen();
        gui.addColor({c:'#2E8B57'}, 'c').name('石绿').onChange(v => {
            PALETTE.shiLv.set(v); generateLandscape();
        });

    </script>
</body>
</html>
