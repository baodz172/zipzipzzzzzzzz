<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>Cyber City Survival - Pro Edition</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <script src="https://cdn.tailwindcss.com"></script>
    <script type="module">
        import { initializeApp } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-app.js";
        import { getAuth, signInAnonymously, onAuthStateChanged, signInWithCustomToken } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-auth.js";
        import { getFirestore, doc, setDoc, getDoc, collection } from "https://www.gstatic.com/firebasejs/11.6.1/firebase-firestore.js";

        // Cấu hình Firebase
        const firebaseConfig = JSON.parse(__firebase_config);
        const app = initializeApp(firebaseConfig);
        const auth = getAuth(app);
        const db = getFirestore(app);
        const appId = typeof __app_id !== 'undefined' ? __app_id : 'cyber-survival-pro';

        // Xuất các hàm cần thiết ra window để sử dụng trong script game
        window.gameStorage = { 
            db, auth, appId, doc, setDoc, getDoc, 
            signInWithCustomToken, signInAnonymously, onAuthStateChanged 
        };
        
        // Kích hoạt sự kiện sẵn sàng để khởi động game logic
        window.dispatchEvent(new Event('firebase-ready'));
    </script>
    <style>
        body { margin: 0; overflow: hidden; background: #000; font-family: 'Inter', sans-serif; touch-action: none; user-select: none; }
        canvas { display: block; }
        #ui { position: absolute; top: 20px; left: 20px; color: white; pointer-events: none; z-index: 10; width: 250px; }
        .stat-card { background: rgba(0,0,0,0.8); border-left: 4px solid #00f2ff; padding: 12px; border-radius: 8px; margin-bottom: 8px; backdrop-filter: blur(8px); box-shadow: 0 4px 15px rgba(0,0,0,0.5); }
        
        #shop-ui { 
            position: absolute; top: 50%; right: 20px; transform: translateY(-50%); 
            background: rgba(10, 10, 30, 0.95); border: 2px solid #00f2ff; 
            padding: 20px; border-radius: 15px; color: white; width: 280px;
            display: none; flex-direction: column; gap: 8px; pointer-events: auto;
            max-height: 80vh; overflow-y: auto; z-index: 100;
        }
        .shop-item { 
            background: rgba(255,255,255,0.05); padding: 12px; border-radius: 10px; 
            cursor: pointer; transition: 0.2s; border: 1px solid transparent;
            display: flex; justify-content: space-between; align-items: center;
        }
        .shop-item:hover { background: rgba(0, 242, 255, 0.15); border-color: #00f2ff; transform: scale(1.02); }
        
        #joystick-area { position: absolute; bottom: 50px; left: 50px; width: 140px; height: 140px; background: rgba(255,255,255,0.05); border: 2px solid rgba(0,242,255,0.3); border-radius: 50%; z-index: 50; }
        #joystick-stick { position: absolute; top: 50%; left: 50%; width: 60px; height: 60px; background: radial-gradient(circle, #00f2ff, #0088ff); border-radius: 50%; transform: translate(-50%, -50%); pointer-events: none; box-shadow: 0 0 20px #00f2ff; }
        #shoot-btn { position: absolute; bottom: 60px; right: 60px; width: 110px; height: 110px; background: linear-gradient(135deg, #ff0055, #aa0033); border-radius: 50%; display: flex; align-items: center; justify-content: center; color: white; font-weight: 900; z-index: 50; box-shadow: 0 0 30px rgba(255,0,85,0.6); cursor: pointer; text-shadow: 0 2px 4px rgba(0,0,0,0.5); }
        
        #shop-toggle { position: absolute; top: 20px; right: 20px; padding: 12px 25px; background: linear-gradient(90deg, #ffaa00, #ff8800); border-radius: 8px; color: black; font-weight: bold; cursor: pointer; pointer-events: auto; box-shadow: 0 4px 10px rgba(255,170,0,0.4); }
        #save-status { position: absolute; bottom: 20px; left: 50%; transform: translateX(-50%); color: #00f2ff; font-size: 12px; opacity: 0; transition: 0.5s; }

        .overlay { position: fixed; inset: 0; background: rgba(0,0,0,0.95); display: none; flex-direction: column; align-items: center; justify-content: center; color: white; z-index: 200; }
        .btn-action { margin-top: 20px; padding: 15px 40px; background: #00f2ff; color: black; font-weight: bold; border-radius: 50px; cursor: pointer; transition: 0.3s; }
        .btn-action:hover { transform: scale(1.1); box-shadow: 0 0 20px #00f2ff; }
    </style>
</head>
<body>

    <div id="ui">
        <div class="stat-card">
            <div class="flex justify-between font-bold"><span class="text-cyan-400">SINH MỆNH</span><span id="hp-val">100</span></div>
            <div class="h-2 w-full bg-gray-800 mt-2 rounded-full overflow-hidden">
                <div id="hp-bar" class="h-full bg-gradient-to-r from-green-500 to-emerald-400 transition-all" style="width: 100%"></div>
            </div>
        </div>
        <div class="stat-card">
            <div class="text-yellow-400 font-bold flex justify-between"><span>XU CYBER</span><span id="score-val">0</span></div>
        </div>
        <div id="boss-warning" class="hidden stat-card border-red-500 bg-red-900/20 text-red-500 font-black animate-pulse text-center">BOSS ĐANG XUẤT HIỆN!</div>
    </div>

    <div id="shop-toggle" onclick="toggleShop()">SHOP (P)</div>
    <div id="save-status">Đã lưu dữ liệu đám mây...</div>

    <div id="shop-ui">
        <h2 class="text-center font-black text-2xl border-b border-cyan-800 pb-3 mb-3 text-cyan-400">CYBER SHOP</h2>
        <div class="shop-item" onclick="buyUpgrade('damage')">
            <span>Sát Thương (+10)</span>
            <b class="text-yellow-400" id="cost-damage">100</b>
        </div>
        <div class="shop-item" onclick="buyUpgrade('speed')">
            <span>Tốc Chạy (+15%)</span>
            <b class="text-yellow-400" id="cost-speed">150</b>
        </div>
        <div class="shop-item" onclick="buyUpgrade('fireRate')">
            <span>Tốc Bắn (+20%)</span>
            <b class="text-yellow-400" id="cost-fireRate">200</b>
        </div>
        <div class="shop-item" onclick="buyUpgrade('range')">
            <span>Tầm Bắn (+20)</span>
            <b class="text-yellow-400" id="cost-range">120</b>
        </div>
        <div class="shop-item" onclick="buyUpgrade('maxHp')">
            <span>Máu Tối Đa (+50)</span>
            <b class="text-yellow-400" id="cost-maxHp">300</b>
        </div>
        <div class="shop-item" onclick="buyUpgrade('heal')">
            <span>Hồi Phục (Full)</span>
            <b class="text-yellow-400">80</b>
        </div>
        <button class="mt-4 bg-red-600 hover:bg-red-700 p-3 rounded-xl font-bold transition-colors" onclick="toggleShop()">QUAY LẠI CHIẾN TRƯỜNG</button>
    </div>

    <div id="joystick-area"><div id="joystick-stick"></div></div>
    <div id="shoot-btn">KHAI HỎA</div>

    <div id="lose-screen" class="overlay">
        <h1 class="text-6xl font-black text-red-600 drop-shadow-lg">GAME OVER</h1>
        <p class="mt-4 text-xl opacity-80">Dữ liệu cuối cùng: <span id="final-score" class="text-yellow-400">0</span> Xu</p>
        <div class="btn-action" onclick="location.reload()">HỒI SINH TẠI TRẠM</div>
    </div>

    <script>
        // --- CORE GAME LOGIC ---
        let scene, camera, renderer, clock, player;
        let obstacles = [], enemies = [], bullets = [], enemyBullets = [], items = [], particles = [];
        let score = 0, hp = 100, maxHp = 100, isGameOver = false;
        let playerStats = { damage: 15, speed: 0.28, fireRate: 400, range: 60 };
        let upgradeCosts = { damage: 100, speed: 150, fireRate: 200, range: 120, maxHp: 300 };
        let bossActive = false, bossLevel = 1;
        let keys = {};
        let moveInput = { x: 0, y: 0 };
        let user = null;

        // Đợi Firebase khởi tạo xong
        window.addEventListener('firebase-ready', () => {
            init();
        });

        async function init() {
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x020208);
            scene.fog = new THREE.FogExp2(0x020208, 0.01);

            camera = new THREE.PerspectiveCamera(70, window.innerWidth / window.innerHeight, 0.1, 1000);
            renderer = new THREE.WebGLRenderer({ antialias: true, alpha: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.shadowMap.enabled = true;
            document.body.appendChild(renderer.domElement);

            clock = new THREE.Clock();

            // Ánh sáng
            scene.add(new THREE.AmbientLight(0x404040, 0.5));
            const sun = new THREE.DirectionalLight(0x00f2ff, 1.2);
            sun.position.set(50, 150, 50);
            sun.castShadow = true;
            scene.add(sun);

            // Sàn nhà
            const floorGeo = new THREE.PlaneGeometry(1000, 1000);
            const floorMat = new THREE.MeshStandardMaterial({ color: 0x0a0a0a, roughness: 0.9 });
            const floor = new THREE.Mesh(floorGeo, floorMat);
            floor.rotation.x = -Math.PI / 2;
            floor.receiveShadow = true;
            scene.add(floor);

            const grid = new THREE.GridHelper(1000, 100, 0x00f2ff, 0x001122);
            scene.add(grid);

            // Nhân vật
            player = new THREE.Group();
            const pBody = new THREE.Mesh(new THREE.BoxGeometry(1, 2, 0.8), new THREE.MeshStandardMaterial({color: 0x00f2ff, emissive: 0x004455}));
            pBody.position.y = 1;
            player.add(pBody);
            scene.add(player);

            // Tạo bản đồ
            for(let i=0; i<100; i++) {
                createComplexBuilding((Math.random()-0.5)*800, (Math.random()-0.5)*800);
            }

            setupControls();
            await setupStorage();
            
            animate();
            
            // Bộ tạo quái và vật phẩm
            setInterval(() => { if(!isGameOver && enemies.length < 60) spawnEnemy(); }, 2000);
            setInterval(() => { if(!isGameOver && items.length < 50) spawnItem(); }, 3000);
            setInterval(saveProgress, 10000); // Tự động lưu sau mỗi 10 giây
        }

        async function setupStorage() {
            const { auth, db, doc, getDoc, signInWithCustomToken, signInAnonymously, onAuthStateChanged, appId } = window.gameStorage;
            
            try {
                if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
                    await signInWithCustomToken(auth, __initial_auth_token);
                } else {
                    await signInAnonymously(auth);
                }

                onAuthStateChanged(auth, async (u) => {
                    if (u) {
                        user = u;
                        // RULE 1: Cấu trúc phân đoạn chẵn cho đường dẫn Document (6 segments)
                        // /artifacts/{appId}/users/{userId}/data/saveData
                        const docRef = doc(db, 'artifacts', appId, 'users', user.uid, 'data', 'saveData');
                        const docSnap = await getDoc(docRef);
                        if (docSnap.exists()) {
                            const data = docSnap.data();
                            score = data.score || 0;
                            hp = data.hp || 100;
                            maxHp = data.maxHp || 100;
                            playerStats = data.playerStats || playerStats;
                            upgradeCosts = data.upgradeCosts || upgradeCosts;
                            if(data.pos) player.position.set(data.pos.x, 1, data.pos.z);
                            updateUI();
                        }
                    }
                });
            } catch (err) {
                console.error("Lỗi thiết lập lưu trữ:", err);
            }
        }

        async function saveProgress() {
            if (!user) return;
            const { db, appId, doc, setDoc } = window.gameStorage;
            try {
                // Sử dụng đường dẫn có cấu trúc chẵn (6 segments)
                await setDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'data', 'saveData'), {
                    score, hp, maxHp, playerStats, upgradeCosts,
                    pos: { x: player.position.x, z: player.position.z },
                    timestamp: Date.now()
                });
                const status = document.getElementById('save-status');
                status.style.opacity = 1;
                setTimeout(() => status.style.opacity = 0, 2000);
            } catch (e) { console.error("Lỗi khi lưu dữ liệu:", e); }
        }

        function createComplexBuilding(x, z) {
            const floors = Math.floor(Math.random() * 6) + 2;
            const size = 10 + Math.random() * 8;
            const group = new THREE.Group();
            
            for(let f=0; f<floors; f++) {
                const floorGeo = new THREE.BoxGeometry(size, 0.5, size);
                const mat = new THREE.MeshStandardMaterial({ color: 0x1a1a1a, metalness: 0.5 });
                const mesh = new THREE.Mesh(floorGeo, mat);
                mesh.position.y = f * 4;
                mesh.receiveShadow = true;
                group.add(mesh);

                const ramp = new THREE.Mesh(new THREE.BoxGeometry(3, 0.2, 5.7), mat);
                ramp.position.set(size/2 - 1.5, f * 4 + 2, size/2 - 1.5);
                ramp.rotation.x = -Math.PI / 4.5;
                group.add(ramp);
            }
            group.position.set(x, 0, z);
            scene.add(group);
            obstacles.push(group);
        }

        function spawnEnemy(isBoss = false) {
            const types = [
                { color: 0xff3300, name: 'Runner', speed: 0.18, hp: 40, size: 1 },
                { color: 0xaa00ff, name: 'Tank', speed: 0.08, hp: 150, size: 1.8 },
                { color: 0x00ffaa, name: 'Shooter', speed: 0.12, hp: 30, size: 1.2 },
                { color: 0xffffff, name: 'Ghost', speed: 0.25, hp: 20, size: 0.8 },
                { color: 0xffff00, name: 'Spitter', speed: 0.1, hp: 60, size: 1.3 }
            ];
            
            const type = isBoss ? { color: 0xff0000, hp: 1000 * bossLevel, speed: 0.05, size: 5 } : types[Math.floor(Math.random() * types.length)];
            const group = new THREE.Group();
            const scale = type.size;
            
            const body = new THREE.Mesh(new THREE.BoxGeometry(scale, 2 * scale, scale), new THREE.MeshStandardMaterial({ color: type.color, emissive: type.color, emissiveIntensity: 0.2 }));
            body.position.y = scale;
            group.add(body);
            
            const angle = Math.random() * Math.PI * 2;
            const dist = 60 + Math.random() * 40;
            group.position.set(player.position.x + Math.cos(angle)*dist, 0, player.position.z + Math.sin(angle)*dist);
            
            group.userData = { hp: type.hp, maxHp: type.hp, isBoss: isBoss, speed: type.speed, lastShot: 0, type: type.name };
            scene.add(group);
            enemies.push(group);
            
            if(isBoss) {
                bossActive = true;
                document.getElementById('boss-warning').classList.remove('hidden');
            }
        }

        function spawnItem() {
            const x = (Math.random()-0.5)*900;
            const z = (Math.random()-0.5)*900;
            const types = ['heal', 'gold', 'buff'];
            const type = types[Math.floor(Math.random() * types.length)];
            const mesh = new THREE.Mesh(new THREE.OctahedronGeometry(0.6), new THREE.MeshBasicMaterial({ color: type === 'heal' ? 0x00ff00 : (type === 'gold' ? 0xffaa00 : 0x00f2ff) }));
            mesh.position.set(x, 1, z);
            mesh.userData = { type };
            scene.add(mesh);
            items.push(mesh);
        }

        window.buyUpgrade = (type) => {
            if(type === 'heal' && score >= 80) { score -= 80; hp = maxHp; }
            else if(score >= upgradeCosts[type]) {
                score -= upgradeCosts[type];
                if(type === 'damage') playerStats.damage += 10;
                if(type === 'speed') playerStats.speed *= 1.15;
                if(type === 'fireRate') playerStats.fireRate *= 0.8;
                if(type === 'range') playerStats.range += 20;
                if(type === 'maxHp') { maxHp += 50; hp += 50; }
                
                upgradeCosts[type] = Math.floor(upgradeCosts[type] * 1.6);
                const el = document.getElementById(`cost-${type}`);
                if(el) el.innerText = upgradeCosts[type];
            }
            updateUI();
            saveProgress();
        };

        window.toggleShop = () => {
            const shop = document.getElementById('shop-ui');
            shop.style.display = shop.style.display === 'flex' ? 'none' : 'flex';
        };

        function setupControls() {
            window.addEventListener('keydown', e => { 
                keys[e.code] = true; 
                if(e.code === 'KeyP') toggleShop();
                if(e.code === 'Space') shoot();
            });
            window.addEventListener('keyup', e => keys[e.code] = false);

            const joyArea = document.getElementById('joystick-area');
            const joyStick = document.getElementById('joystick-stick');
            const handleMove = (e) => {
                const clientX = e.touches ? e.touches[0].clientX : e.clientX;
                const clientY = e.touches ? e.touches[0].clientY : e.clientY;
                const rect = joyArea.getBoundingClientRect();
                const centerX = rect.left + rect.width/2;
                const centerY = rect.top + rect.height/2;
                const dx = clientX - centerX;
                const dy = clientY - centerY;
                const dist = Math.min(70, Math.hypot(dx, dy));
                const angle = Math.atan2(dy, dx);
                joyStick.style.transform = `translate(calc(-50% + ${Math.cos(angle)*dist}px), calc(-50% + ${Math.sin(angle)*dist}px))`;
                moveInput.x = Math.cos(angle) * (dist/70);
                moveInput.y = Math.sin(angle) * (dist/70);
            };
            joyArea.addEventListener('pointermove', handleMove);
            joyArea.addEventListener('touchstart', (e) => e.preventDefault());
            joyArea.addEventListener('touchmove', (e) => { e.preventDefault(); handleMove(e); });
            const resetJoy = () => { joyStick.style.transform = `translate(-50%, -50%)`; moveInput = { x: 0, y: 0 }; };
            joyArea.addEventListener('pointerup', resetJoy);
            joyArea.addEventListener('touchend', resetJoy);
            document.getElementById('shoot-btn').addEventListener('pointerdown', shoot);
        }

        let lastFire = 0;
        function shoot() {
            if(Date.now() - lastFire < playerStats.fireRate) return;
            lastFire = Date.now();
            const b = new THREE.Mesh(new THREE.SphereGeometry(0.35), new THREE.MeshBasicMaterial({ color: 0x00f2ff }));
            b.position.copy(player.position).y += 1.2;
            const dir = new THREE.Vector3(0, 0, -1).applyQuaternion(player.quaternion);
            b.userData = { velocity: dir.multiplyScalar(1.5), life: playerStats.range, damage: playerStats.damage };
            scene.add(b);
            bullets.push(b);
        }

        function enemyShoot(en) {
            const b = new THREE.Mesh(new THREE.SphereGeometry(0.35), new THREE.MeshBasicMaterial({ color: 0xff0055 }));
            b.position.copy(en.position).y += en.userData.isBoss ? 4 : 1.2;
            const dir = player.position.clone().sub(en.position).normalize();
            b.userData = { velocity: dir.multiplyScalar(0.7), life: 80 };
            scene.add(b);
            enemyBullets.push(b);
        }

        function updateUI() {
            document.getElementById('hp-val').innerText = Math.ceil(hp);
            document.getElementById('hp-bar').style.width = (hp/maxHp * 100) + '%';
            document.getElementById('score-val').innerText = score;
        }

        function animate() {
            if(isGameOver) return;
            requestAnimationFrame(animate);
            const delta = clock.getDelta();

            let mx = moveInput.x, mz = moveInput.y;
            if(keys['ArrowLeft'] || keys['KeyA']) mx = -1;
            if(keys['ArrowRight'] || keys['KeyD']) mx = 1;
            if(keys['ArrowUp'] || keys['KeyW']) mz = -1;
            if(keys['ArrowDown'] || keys['KeyS']) mz = 1;

            if(mx !== 0 || mz !== 0) {
                const moveVec = new THREE.Vector3(mx, 0, mz).normalize().multiplyScalar(playerStats.speed);
                player.position.add(moveVec);
                player.rotation.y = Math.atan2(mx, mz);
                
                let onObj = false;
                obstacles.forEach(obj => {
                    const box = new THREE.Box3().setFromObject(obj);
                    if(box.containsPoint(player.position.clone().add(new THREE.Vector3(0, 0.5, 0)))) {
                        player.position.y = box.max.y;
                        onObj = true;
                    }
                });
                if(!onObj && player.position.y > 0) player.position.y -= 0.15;
            }

            camera.position.lerp(new THREE.Vector3(player.position.x, player.position.y + 18, player.position.z + 20), 0.1);
            camera.lookAt(player.position);

            bullets.forEach((b, i) => {
                b.position.add(b.userData.velocity);
                b.userData.life--;
                enemies.forEach((en, ei) => {
                    if(b.position.distanceTo(en.position) < (en.userData.isBoss ? 5 : 2)) {
                        en.userData.hp -= b.userData.damage;
                        b.userData.life = 0;
                        if(en.userData.hp <= 0) {
                            score += en.userData.isBoss ? (500 * bossLevel) : 25;
                            if(en.userData.isBoss) { bossActive = false; bossLevel++; document.getElementById('boss-warning').classList.add('hidden'); }
                            scene.remove(en); enemies.splice(ei, 1);
                            updateUI();
                        }
                    }
                });
                if(b.userData.life <= 0) { scene.remove(b); bullets.splice(i, 1); }
            });

            enemyBullets.forEach((b, i) => {
                b.position.add(b.userData.velocity);
                b.userData.life--;
                if(b.position.distanceTo(player.position) < 1.8) {
                    hp -= 12; updateUI(); b.userData.life = 0;
                    if(hp <= 0) { isGameOver = true; document.getElementById('lose-screen').style.display = 'flex'; document.getElementById('final-score').innerText = score; saveProgress(); }
                }
                if(b.userData.life <= 0) { scene.remove(b); enemyBullets.splice(i, 1); }
            });

            enemies.forEach(en => {
                const dist = en.position.distanceTo(player.position);
                if(dist < 70) {
                    const dir = player.position.clone().sub(en.position).normalize();
                    en.position.add(dir.multiplyScalar(en.userData.speed));
                    if(Date.now() - en.userData.lastShot > 1800) { enemyShoot(en); en.userData.lastShot = Date.now(); }
                    
                    obstacles.forEach(obj => {
                        const box = new THREE.Box3().setFromObject(obj);
                        if(box.containsPoint(en.position.clone().add(new THREE.Vector3(0, 0.5, 0)))) en.position.y = box.max.y;
                    });
                }
            });

            items.forEach((it, i) => {
                it.rotation.y += 0.05;
                if(it.position.distanceTo(player.position) < 2.5) {
                    if(it.userData.type === 'gold') score += 100;
                    if(it.userData.type === 'heal') hp = Math.min(maxHp, hp + 40);
                    scene.remove(it); items.splice(i, 1);
                    updateUI();
                }
            });

            if(score > 0 && score % (2000 * bossLevel) === 0 && !bossActive) spawnEnemy(true);

            renderer.render(scene, camera);
        }

        window.addEventListener('resize', () => {
            camera.aspect = window.innerWidth / window.innerHeight; camera.updateProjectionMatrix();
            renderer.setSize(window.innerWidth, window.innerHeight);
        });
    </script>
</body>
</html>
