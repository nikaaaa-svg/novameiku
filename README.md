<!DOCTYPE html>
<html>
<head>
    <title>迷宮クリエイター：マイリスト機能</title>
    <style>
        body { font-family: sans-serif; text-align: center; background: #eef2f3; margin: 0; padding: 20px; }
        #game-container { display: inline-block; background: white; padding: 20px; border-radius: 15px; box-shadow: 0 10px 25px rgba(0,0,0,0.1); width: 600px; }
        .row { display: flex; justify-content: center; }
        .cell { width: 35px; height: 35px; border: 1px solid #eee; cursor: pointer; position: relative; }
        .wall { background: #333; }
        .road { background: #fff; }
        .player { background: #007bff; border-radius: 50%; border: 2px solid white; box-sizing: border-box; width: 100%; height: 100%; position: absolute; top: 0; left: 0; z-index: 2; }
        .goal { background: #ff4757; }
        .screen { display: none; }
        .active { display: block; }
        button { margin: 5px; padding: 8px 12px; cursor: pointer; border-radius: 5px; border: none; background: #007bff; color: white; }
        button:hover { opacity: 0.8; }
        .toolbar { background: #f8f9fa; padding: 10px; border-radius: 8px; margin-bottom: 10px; }
        .selected { background: #ffaa00 !important; color: black; }
        /* リストのスタイル */
        #maze-list { text-align: left; background: #fdfdfd; border: 1px solid #ddd; padding: 10px; border-radius: 5px; margin-top: 10px; max-height: 200px; overflow-y: auto; }
        .maze-item { display: flex; justify-content: space-between; padding: 5px; border-bottom: 1px solid #eee; }
        .maze-item:hover { background: #f0f0f0; }
    </style>
</head>
<body>

    <div id="game-container">
        <div id="home-screen" class="screen active">
            <h1>迷宮クリエイター</h1>
            <button onclick="showScreen('edit-screen')">新しく作る</button>
            <hr>
            <h3>マイ迷宮リスト</h3>
            <div id="maze-list">保存された迷宮はありません</div>
        </div>

        <div id="edit-screen" class="screen">
            <h2>設計モード</h2>
            <div class="toolbar">
                <input type="text" id="maze-name" placeholder="迷宮の名前を入力">
                <button onclick="saveMaze()" style="background:#28a745;">PCに保存</button>
                <hr>
                フロア: <button onclick="changeEditFloor(-1)">▲</button> <span id="edit-floor-num">1</span>F <button onclick="changeEditFloor(1)">▼</button>
                <button id="btn-wall" onclick="setMode('wall')" class="selected">壁・道</button>
                <button id="btn-start" onclick="setMode('start')">開始点</button>
                <button id="btn-goal" onclick="setMode('goal')">階段</button>
            </div>
            <div id="edit-board"></div>
            <button onclick="startTest()" style="background:#ffaa00; margin-top:10px;">テストプレイ</button>
            <button onclick="showScreen('home-screen')" style="background:#666;">ホームへ</button>
        </div>

        <div id="play-screen" class="screen">
            <h2 id="playing-title">テスト中</h2>
            <div id="play-floor-num-disp">1F</div>
            <div id="play-board"></div>
            <button id="uploadBtn" style="display:none; background: #28a745;" onclick="alert('世界に投稿されました！（サーバー連携は次のステップへ）')">★ 投稿する</button>
            <button onclick="showScreen('edit-screen')" style="background:#666;">戻る</button>
        </div>
    </div>

    <script>
        // 初期の空迷宮
        function createEmptyMaze() {
            return Array(7).fill().map(() => Array(10).fill(0));
        }

        let mazes = [createEmptyMaze(), createEmptyMaze(), createEmptyMaze()];
        let playerStart = { x: 0, y: 0 };
        let player = { x: 0, y: 0 };
        let currentEditFloor = 0;
        let currentPlayFloor = 0;
        let editMode = 'wall';

        // --- 保存・読み込み機能 ---
        function saveMaze() {
            const name = document.getElementById('maze-name').value || "ななしの迷宮";
            const mazeData = {
                name: name,
                data: mazes,
                start: playerStart,
                date: new Date().toLocaleString()
            };
            
            // ローカルストレージ（ブラウザの記憶領域）から既存リスト取得
            let list = JSON.parse(localStorage.getItem('myMazes') || '[]');
            list.push(mazeData);
            localStorage.setItem('myMazes', JSON.stringify(list));
            
            alert("「" + name + "」を保存しました！");
            updateMazeList();
        }

        function updateMazeList() {
            const listArea = document.getElementById('maze-list');
            const list = JSON.parse(localStorage.getItem('myMazes') || '[]');
            
            if (list.length === 0) {
                listArea.innerHTML = "保存された迷宮はありません";
                return;
            }

            listArea.innerHTML = '';
            list.forEach((item, index) => {
                const div = document.createElement('div');
                div.className = 'maze-item';
                div.innerHTML = `
                    <span>${item.name} (${item.date})</span>
                    <div>
                        <button onclick="loadMaze(${index})" style="background:#28a745">開く</button>
                        <button onclick="deleteMaze(${index})" style="background:#dc3545">消す</button>
                    </div>
                `;
                listArea.appendChild(div);
            });
        }

        function loadMaze(index) {
            const list = JSON.parse(localStorage.getItem('myMazes') || '[]');
            const item = list[index];
            mazes = item.data;
            playerStart = item.start;
            document.getElementById('maze-name').value = item.name;
            showScreen('edit-screen');
            alert(item.name + " を読み込みました");
        }

        function deleteMaze(index) {
            if(!confirm("本当に消しますか？")) return;
            let list = JSON.parse(localStorage.getItem('myMazes') || '[]');
            list.splice(index, 1);
            localStorage.setItem('myMazes', JSON.stringify(list));
            updateMazeList();
        }

        // --- 基本システム ---
        function setMode(mode) {
            editMode = mode;
            ['btn-wall', 'btn-start', 'btn-goal'].forEach(id => {
                document.getElementById(id).classList.toggle('selected', id === 'btn-' + mode);
            });
        }

        function changeEditFloor(delta) {
            currentEditFloor = Math.max(0, Math.min(mazes.length - 1, currentEditFloor + delta));
            document.getElementById('edit-floor-num').innerText = currentEditFloor + 1;
            drawBoard('edit-board', true);
        }

        function showScreen(screenId) {
            document.querySelectorAll('.screen').forEach(s => s.classList.remove('active'));
            document.getElementById(screenId).classList.add('active');
            if(screenId === 'edit-screen') drawBoard('edit-board', true);
            if(screenId === 'home-screen') updateMazeList();
        }

        function drawBoard(targetId, isEdit) {
            const board = document.getElementById(targetId);
            board.innerHTML = '';
            const activeMaze = isEdit ? mazes[currentEditFloor] : mazes[currentPlayFloor];
            
            activeMaze.forEach((row, y) => {
                const rowDiv = document.createElement('div');
                rowDiv.className = 'row';
                row.forEach((cell, x) => {
                    const cellDiv = document.createElement('div');
                    cellDiv.className = 'cell ' + (cell === 1 ? 'wall' : (cell === 2 ? 'goal' : 'road'));
                    
                    const isPlayerHere = isEdit 
                        ? (currentEditFloor === 0 && x === playerStart.x && y === playerStart.y)
                        : (x === player.x && y === player.y);
                    
                    if (isPlayerHere) {
                        const p = document.createElement('div');
                        p.className = 'player';
                        cellDiv.appendChild(p);
                    }
                    
                    if (isEdit) {
                        cellDiv.onclick = () => {
                            if (editMode === 'wall') {
                                if (activeMaze[y][x] === 2) return;
                                activeMaze[y][x] = (activeMaze[y][x] === 0) ? 1 : 0;
                            } else if (editMode === 'start' && currentEditFloor === 0) {
                                playerStart = { x, y };
                            } else if (editMode === 'goal') {
                                for(let ry=0; ry<activeMaze.length; ry++) 
                                    for(let rx=0; rx<activeMaze[ry].length; rx++)
                                        if(activeMaze[ry][rx]===2) activeMaze[ry][rx]=0;
                                activeMaze[y][x] = 2;
                            }
                            drawBoard(targetId, true);
                        };
                    }
                    rowDiv.appendChild(cellDiv);
                });
                board.appendChild(rowDiv);
            });
        }

        function startTest() {
            currentPlayFloor = 0;
            player = { ...playerStart };
            document.getElementById('uploadBtn').style.display = 'none';
            document.getElementById('play-floor-num-disp').innerText = "1F";
            showScreen('play-screen');
            drawBoard('play-board', false);
        }

        window.addEventListener('keydown', (e) => {
            if (!document.getElementById('play-screen').classList.contains('active')) return;
            let nextX = player.x, nextY = player.y;
            if (e.key === 'ArrowUp') nextY--;
            if (e.key === 'ArrowDown') nextY++;
            if (e.key === 'ArrowLeft') nextX--;
            if (e.key === 'ArrowRight') nextX++;

            const currentMaze = mazes[currentPlayFloor];
            if (currentMaze[nextY] && currentMaze[nextY][nextX] !== undefined && currentMaze[nextY][nextX] !== 1) {
                player.x = nextX; player.y = nextY;
                if (currentMaze[nextY][nextX] === 2) {
                    if (currentPlayFloor < mazes.length - 1) {
                        alert("次の階へ！");
                        currentPlayFloor++;
                        document.getElementById('play-floor-num-disp').innerText = (currentPlayFloor + 1) + "F";
                    } else {
                        alert("脱出成功！");
                        document.getElementById('uploadBtn').style.display = 'inline-block';
                    }
                }
            }
            drawBoard('play-board', false);
        });

        updateMazeList(); // 最初の一覧表示
    </script>
</body>
</html>
