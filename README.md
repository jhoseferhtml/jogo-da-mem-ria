<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Jogo da Memória</title>
    <style>
        body {
            font-family: 'Arial', sans-serif;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            margin: 0;
            background: linear-gradient(to bottom, #f0f8ff, #e0ffff);
            color: #333;
        }
        h1 {
            margin-bottom: 20px;
            color: #0077cc;
        }
        .game-board {
            display: grid;
            grid-template-columns: repeat(4, 120px);
            gap: 15px;
            margin-bottom: 20px;
        }
        .card {
            width: 120px;
            height: 120px;
            background-color: #f7f7f7;
            display: flex;
            align-items: center;
            justify-content: center;
            font-size: 40px;
            cursor: pointer;
            border-radius: 10px;
            box-shadow: 0 4px 10px rgba(0, 0, 0, 0.2);
            transition: all 0.3s ease;
        }
        .card.flipped {
            background-color: #fff;
            transform: rotateY(180deg);
            box-shadow: 0 8px 20px rgba(0, 0, 0, 0.3);
        }
        .card.matched {
            background-color: #aaffaa;
            pointer-events: none;
        }
        #info {
            margin-bottom: 20px;
            font-size: 20px;
        }
        #timer {
            color: red;
        }
        #reset, #next-level, #pause {
            padding: 10px 20px;
            font-size: 16px;
            background-color: #0077cc;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
            transition: background-color 0.3s ease;
            margin-top: 10px;
        }
        #reset:hover, #next-level:hover, #pause:hover {
            background-color: #005fa3;
        }
        #next-level {
            display: none; /* Inicialmente oculto */
        }
        .modal {
            display: none;
            position: fixed;
            top: 0;
            left: 0;
            right: 0;
            bottom: 0;
            background: rgba(0, 0, 0, 0.7);
            color: white;
            justify-content: center;
            align-items: center;
            font-size: 24px;
            z-index: 10;
        }
        .modal-content {
            background: #333;
            padding: 20px;
            border-radius: 10px;
            text-align: center;
        }
        .modal button {
            background: #0077cc;
            border: none;
            padding: 10px 15px;
            border-radius: 5px;
            cursor: pointer;
            margin-top: 10px;
        }
        .modal button:hover {
            background: #005fa3;
        }
        .score-table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 20px;
        }
        .score-table th, .score-table td {
            border: 1px solid #0077cc;
            padding: 10px;
            text-align: center;
            font-size: 18px;
        }
        .score-table th {
            background-color: #0077cc;
            color: white;
        }
    </style>
</head>
<body>
    <h1>Jogo da Memória</h1>
    <div id="info">
        <span>Nível: <span id="level-display">1</span></span>
        <span>Pontuação: <span id="score">0</span></span>
        <span id="attempts">Tentativas Restantes: <span id="remaining-attempts">15</span></span>
        <span id="timer">Tempo: <span id="time">30</span> segundos</span>
    </div>
    <div class="game-board" id="game-board"></div>
    <button id="reset">Reiniciar Jogo</button>
    <button id="pause">Pausar</button>
    <button id="next-level">Próximo Nível</button>

    <div class="modal" id="modal">
        <div class="modal-content">
            <p id="modal-message"></p>
            <table class="score-table">
                <thead>
                    <tr>
                        <th>Pontuação Atual</th>
                        <th>Maior Pontuação</th>
                        <th>Nível Máximo Alcançado</th>
                    </tr>
                </thead>
                <tbody>
                    <tr>
                        <td id="current-score">0</td>
                        <td id="high-score">0</td>
                        <td id="max-level">0</td>
                    </tr>
                </tbody>
            </table>
            <button id="restart-game">Reiniciar Jogo</button>
            <button id="close-modal">Fechar</button>
        </div>
    </div>
    
    <script>
        const totalLevels = 20;
        let currentLevel = 0;
        let score = 0;
        let highScore = localStorage.getItem('highScore') || 0;
        let maxLevelReached = localStorage.getItem('maxLevel') || 0;
        let time = 30;
        let interval;
        let firstCard = null;
        let secondCard = null;
        let lockBoard = false;
        let matchedCards = 0;
        let remainingAttempts = 15;
        let isPaused = false;

        const gameBoard = document.getElementById('game-board');
        const scoreDisplay = document.getElementById('score');
        const timeDisplay = document.getElementById('time');
        const attemptsDisplay = document.getElementById('remaining-attempts');
        const nextLevelButton = document.getElementById('next-level');
        const pauseButton = document.getElementById('pause');
        const modal = document.getElementById('modal');
        const modalMessage = document.getElementById('modal-message');
        const closeModalButton = document.getElementById('close-modal');
        const restartGameButton = document.getElementById('restart-game');
        const currentScoreDisplay = document.getElementById('current-score');
        const highScoreDisplay = document.getElementById('high-score');
        const maxLevelDisplay = document.getElementById('max-level');
        const levelDisplay = document.getElementById('level-display');

        const emojis = ['🐶', '🐱', '🐰', '🐻', '🦁', '🐼', '🐨', '🐸', '🦄', '🦋', '🌟', '🍀', '🌈', '🌻', '🎉', '💎', '🌼', '🍉', '🌺', '⚡'];

        function getCardValues(numCards) {
            const cardValues = [];
            for (let i = 0; i < numCards / 2; i++) {
                cardValues.push(emojis[i], emojis[i]);
            }
            return cardValues;
        }

        function shuffle(array) {
            array.sort(() => Math.random() - 0.5);
        }

        function createBoard() {
            const numCards = (currentLevel + 2) * 2;
            const cardValues = getCardValues(numCards);
            shuffle(cardValues);
            gameBoard.innerHTML = '';
            cardValues.forEach(value => {
                const card = document.createElement('div');
                card.classList.add('card');
                card.setAttribute('data-value', value);
                card.addEventListener('click', flipCard);
                gameBoard.appendChild(card);
            });
            matchedCards = 0;
            remainingAttempts = 15;
            attemptsDisplay.textContent = remainingAttempts;
            nextLevelButton.style.display = 'none'; // Esconde botão próximo nível
            startTimer();
            levelDisplay.textContent = currentLevel + 1; // Atualiza o nível atual
        }

        function flipCard() {
            if (lockBoard || isPaused || this.classList.contains('flipped') || this.classList.contains('matched')) return;

            this.classList.add('flipped');
            this.textContent = this.getAttribute('data-value');

            if (!firstCard) {
                firstCard = this;
            } else {
                secondCard = this;
                lockBoard = true;

                checkForMatch();
            }
        }

        function checkForMatch() {
            if (firstCard.getAttribute('data-value') === secondCard.getAttribute('data-value')) {
                firstCard.classList.add('matched');
                secondCard.classList.add('matched');
                matchedCards += 2;
                score += 10;
                scoreDisplay.textContent = score;
                resetTurn();

                if (matchedCards === gameBoard.children.length) {
                    if (currentLevel + 1 === totalLevels) {
                        endGame('Você completou todos os níveis! Fim de jogo.');
                    } else {
                        nextLevelButton.style.display = 'block';
                    }
                }
            } else {
                remainingAttempts--;
                attemptsDisplay.textContent = remainingAttempts;
                setTimeout(() => {
                    firstCard.classList.remove('flipped');
                    secondCard.classList.remove('flipped');
                    firstCard.textContent = '';
                    secondCard.textContent = '';
                    resetTurn();
                }, 1000);

                if (remainingAttempts <= 0) {
                    endGame('Fim de jogo! Você esgotou suas tentativas.');
                }
            }
        }

        function resetTurn() {
            firstCard = null;
            secondCard = null;
            lockBoard = false;
        }

        function startTimer() {
            time = 30;
            timeDisplay.textContent = time;
            clearInterval(interval);
            interval = setInterval(() => {
                time--;
                timeDisplay.textContent = time;

                if (time <= 0) {
                    clearInterval(interval);
                    endGame('O tempo acabou! Fim de jogo.');
                }
            }, 1000);
        }

        function nextLevel() {
            currentLevel++;
            createBoard();
        }

        function endGame(message) {
            if (score > highScore) {
                highScore = score;
                localStorage.setItem('highScore', highScore);
            }
            if (currentLevel + 1 > maxLevelReached) {
                maxLevelReached = currentLevel + 1;
                localStorage.setItem('maxLevel', maxLevelReached);
            }

            modalMessage.textContent = message;
            currentScoreDisplay.textContent = score;
            highScoreDisplay.textContent = highScore;
            maxLevelDisplay.textContent = maxLevelReached;
            modal.style.display = 'flex';
            clearInterval(interval);
        }

        function closeModal() {
            modal.style.display = 'none';
        }

        function restartGame() {
            score = 0;
            currentLevel = 0;
            createBoard();
            closeModal();
        }

        function pauseGame() {
            if (isPaused) {
                startTimer();
                pauseButton.textContent = 'Pausar';
                isPaused = false;
            } else {
                clearInterval(interval);
                lockBoard = true; // Bloqueia cartas enquanto está pausado
                pauseButton.textContent = 'Continuar';
                isPaused = true;
            }
        }

        // Inicializa o jogo
        createBoard();

        // Event listeners
        closeModalButton.addEventListener('click', closeModal);
        nextLevelButton.addEventListener('click', nextLevel);
        restartGameButton.addEventListener('click', restartGame);
        pauseButton.addEventListener('click', pauseGame);
    </script>
</body>
</html>
