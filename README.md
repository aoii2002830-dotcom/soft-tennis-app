<!DOCTYPE html>
<html lang="ja">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <title>ソフトテニス 前衛ポジションシミュレーター</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            background-color: #121824;
            color: #ffffff;
            font-family: 'Helvetica Neue', Arial, sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            min-height: 100vh;
            overflow: hidden;
        }
        h1 {
            font-size: 16px;
            margin: 10px 0 5px 0;
            font-weight: 600;
        }
        p {
            font-size: 11px;
            color: #ccc;
            margin: 0 0 15px 0;
        }
        /* コートのコンテナ（スマホに収まるサイズに調整） */
        #court-container {
            width: 90vw;
            max-width: 360px;
            height: calc(90vw * (23.77 / 10.97));
            max-height: 600px;
            position: relative;
        }
        svg {
            width: 100%;
            height: 100%;
            background-color: #2e7d32; /* テニスコートの緑 */
            border: 4px solid #fff;
            box-sizing: border-box;
            border-radius: 4px;
            box-shadow: 0 10px 25px rgba(0,0,0,0.5);
        }
        /* ドラッグ可能要素のカーソル指定 */
        .draggable {
            cursor: grab;
        }
        .draggable:active {
            cursor: grabbing;
        }
        .label {
            font-size: 12px;
            fill: #fff;
            font-weight: bold;
            text-anchor: middle;
            pointer-events: none;
        }
    </style>
</head>
<body>

    <h1>ソフトテニス 前衛ポジション学習</h1>
    <p>上の赤い打点（相手後衛）を左右にドラッグしてみよう</p>

    <div id="court-container">
        <svg id="court" viewBox="0 0 1097 2377">
            <line x1="0" y1="698.5" x2="1097" y2="698.5" stroke="white" stroke-width="8" />
            <line x1="0" y1="1678.5" x2="1097" y2="1678.5" stroke="white" stroke-width="8" />
            
            <line x1="548.5" y1="698.5" x2="548.5" y2="1678.5" stroke="white" stroke-width="8" />
            
            <line x1="0" y1="1188.5" x2="1097" y2="1188.5" stroke="white" stroke-width="12" stroke-dasharray="15, 10" opacity="0.7" />

            <polygon id="ball-zone" points="548.5,100 0,2377 1097,2377" fill="rgba(255, 255, 255, 0.15)" stroke="rgba(255,255,255,0.3)" stroke-width="3" stroke-dasharray="10,10" />

            <line id="guide-line" x1="548.5" y1="100" x2="548.5" y2="2200" stroke="#4fc3f7" stroke-width="6" stroke-dasharray="12, 12" />

            <circle cx="548.5" y="2200" r="25" fill="#4fc3f7" stroke="white" stroke-width="4" />
            
            <circle id="volleyer" cx="548.5" y="1450" r="35" fill="#ffb300" stroke="white" stroke-width="5" />
            <text id="volleyer-label" x="548.5" y="1455" class="label" fill="#000">前衛</text>

            <g id="striker-group" class="draggable">
                <circle id="striker" cx="548.5" y="100" r="40" fill="#e53935" stroke="white" stroke-width="5" />
                <text x="548.5" y="105" class="label">打点</text>
            </g>
        </svg>
    </div>

    <script>
        const svg = document.getElementById('court');
        const strikerGroup = document.getElementById('striker-group');
        const striker = document.getElementById('striker');
        const strikerText = strikerGroup.querySelector('text');
        const volleyer = document.getElementById('volleyer');
        const volleyerLabel = document.getElementById('volleyer-label');
        const guideLine = document.getElementById('guide-line');
        const ballZone = document.getElementById('ball-zone');

        // 固定値：自陣後衛の位置
        const defenderX = 548.5;
        const defenderY = 2200;

        let isDragging = false;

        // イベントリスナーの登録（マウス＆スマホ両対応）
        strikerGroup.addEventListener('mousedown', startDrag);
        window.addEventListener('mousemove', drag);
        window.addEventListener('mouseup', endDrag);

        strikerGroup.addEventListener('touchstart', startDrag, { passive: false });
        window.addEventListener('touchmove', drag, { passive: false });
        window.addEventListener('touchend', endDrag);

        function startDrag(e) {
            isDragging = true;
            if(e.cancelable) e.preventDefault();
        }

        function endDrag() {
            isDragging = false;
        }

        function drag(e) {
            if (!isDragging) return;
            if(e.cancelable) e.preventDefault();

            // 座標の取得（マウスかタッチか）
            const clientX = e.touches ? e.touches[0].clientX : e.clientX;
            
            // SVG内のローカル座標に変換
            const rect = svg.getBoundingClientRect();
            const svgWidth = 1097;
            let mouseXInSvg = ((clientX - rect.left) / rect.width) * svgWidth;

            // コートの外に出ないように制限（0 〜 1097）
            if (mouseXInSvg < 0) mouseXInSvg = 0;
            if (mouseXInSvg > 1097) mouseXInSvg = 1097;

            // 相手後衛（赤い点）の位置を更新（Y座標は100で固定）
            const strikerY = 100;
            striker.setAttribute('cx', mouseXInSvg);
            strikerText.setAttribute('x', mouseXInSvg);

            // 補助線の更新
            guideLine.setAttribute('x1', mouseXInSvg);

            // 打球範囲（ポリゴン）の更新
            // 相手の打点から、自分のベースライン両端（0,2377 と 1097,2377）へ広がるエリア
            ballZone.setAttribute('points', `${mouseXInSvg},${strikerY} 0,2377 1097,2377`);

            // ★前衛のポジション自動計算ロジック
            // 相手の打点と、自分のセンターマーク付近(548.5, 2200)を結ぶ直線上に前衛を配置
            // 前衛の定位置（ネットから少し下がった Y=1450 付近）でのX座標を割り出す（内分点）
            const volleyerY = 1450; 
            const ratio = (volleyerY - strikerY) / (defenderY - strikerY);
            const volleyerX = mouseXInSvg + (defenderX - mouseXInSvg) * ratio;

            // 前衛の位置を更新
            volleyer.setAttribute('cx', volleyerX);
            volleyerLabel.setAttribute('x', volleyerX);
        }
    </script>
</body>
</html>
