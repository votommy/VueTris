<script setup lang="ts">
// #region imports
// Vue
import { onBeforeMount, onMounted, ref } from 'vue'

// Firebase
import { initializeApp } from 'https://www.gstatic.com/firebasejs/9.7.0/firebase-app.js';
import { getFirestore, collection, getDocs, addDoc, query, orderBy, limit, serverTimestamp } from 'https://www.gstatic.com/firebasejs/9.7.0/firebase-firestore.js';
import { initializeAppCheck, ReCaptchaV3Provider } from 'https://www.gstatic.com/firebasejs/9.7.0/firebase-app-check.js';
// #endregion imports

// #region types
type TetrominoShape = 'T' | 'J' | 'L' | 'O' | 'S' | 'Z' | 'I' | undefined;
type TetrominoMatrix = number[][] | [number, number][] | [number, number, number][] | [number, number, number, number][]
interface GamePiece {
  position: { x: number; y: number };
  matrix: TetrominoMatrix | undefined;
}
// #endregion types

// #region data
const maincanvas = ref()
const maincontext = ref()
const ghostcanvas = ref()
const ghostcontext = ref()
const bullpencanvas = ref()
const bullpencontext = ref()
const jailcellcanvas = ref()
const jailcellcontext = ref()
const gamepiece: GamePiece = { position: { x: 0, y: 0 }, matrix: undefined }
const bullpenpiece: GamePiece = { position: { x: 0, y: 0 }, matrix: undefined }
const jailcellpiece: GamePiece = { position: { x: 0, y: 0 }, matrix: undefined }
const gamearena = canvas(20, 10)
const ghostarena = canvas(20, 10)
const bullpen = canvas(4, 2)
const jailcell = canvas(4, 2)
const colors = [
    null, 
    '0,     255,    255',   /* I cyan */
    '0,     0,      255',   /* J blue */
    '255,   165,    0',     /* L orange */
    '255,   255,    0',     /* O yellow */
    '0,     128,    0',     /* S green */
    '128,   0,      128',   /* T purple */
    '255,   0,      0'      /* Z red */
]
const bg = ref()
const cube = ref()
const tetris = ref()
const gameOverStatus = ref(false)
const pauseStatus = ref(false)
const standby = ref<TetrominoShape>()
const nextPiece = ref<TetrominoShape>(standby.value)
const currentPiece = ref<TetrominoShape>(nextPiece.value)
const isFirstHold = ref(true)
const heldPiece = ref(currentPiece.value)
const isHeldAlready = ref(false)
const cancelId = ref(0)
const dropCounter = ref(0)
const dropSpeed = ref(1000)
const originalDropSpeed = ref(dropSpeed.value)
const time = ref(0)
const timerValue = ref('00:00')
const playTime = ref(0)
const score = ref(0)
const level = ref(1)
const lines = ref(0)
// #endregion data

// #region methods
function kbcontrols(event) {
    if (event.keyCode === 13) {
        document.querySelector("#startDirection").style.display = "none"
        setInterval(timer, 1000)
        document.removeEventListener('keydown', kbcontrols) //After it's pressed, it's no longer needed. TODO: pause btn
        document.addEventListener('keydown', playercontrols)
        document.addEventListener('keydown', restartListener)
        document.addEventListener("keyup", deactivateBtn)
        document.addEventListener("keydown", pauseListener)

        initiateNewGamePiece(standby.value)
        loadBullpen()
         
        requestAnimationFrame(run)
    }
}
function timer() {
    let date = new Date(0)
    if (gameOverStatus.value === false) {
        if (pauseStatus.value === false) {
            date.setSeconds(++playTime.value)
        }
        else {
            date.setSeconds(playTime.value)
        }
        if (playTime.value < 3600) {
            timerValue.value = date.toISOString().substr(14, 5)
        }
        else {
            timerValue.value = date.toISOString().substr(11, 8)
        }
        
    }
    else {
        date.setSeconds(playTime.value)
        if (playTime.value < 3600) {
            timerValue.value = date.toISOString().substr(14, 5)
        }
        else {
            timerValue.value = date.toISOString().substr(11, 8)
        }
    }
}
function playercontrols(event) {
    switch (event.keyCode) {
        case 37: /* left arrow; move left */
            event.preventDefault()
            shiftShape(-1)
            activateBtn("leftArrow", true)
            break
        case 39: /* right arrow; move right */
            event.preventDefault()
            shiftShape(1)
            activateBtn("rightArrow", true)
            break
        case 32: /* space dropFast() */
            event.preventDefault()
            dropFast()
            activateBtn("spacebar", true)
            break
        case 38: /* up arrow; rotate right */
            event.preventDefault()
            rotateShape(1)
            activateBtn("upArrow", true)
            break
        case 40: /* down arrow; drop piece */
            event.preventDefault()
            dropShape()
            activateBtn("downArrow", true)
            break
        case 88: /* x; rotate right */
            rotateShape(1)
            activateBtn("xBtn", true)
            break
        case 90: /* z; rotate left */
            rotateShape(-1)
            activateBtn("zBtn", true)
            break
        case 16: /* shift; hold piece */
            hold()
            activateBtn("shiftBtn", true)
            break
    }
}
function activateBtn(id: string, activate: boolean) {
    document.querySelector(`#${id}`).style.backgroundColor = activate ? "#AAAAFF" : "#DDD"
}
function restartListener(event) {
    if (event.keyCode === 27) {
        restartGame()
    }
}
function pauseListener(event) {
    if (event.keyCode === 80) {
        pauseToggle()
    }
}
function deactivateBtn(event) { //For the visual on-screen keyboard
    switch (event.keyCode) {
        case 37: /* left arrow; move left */
            activateBtn("leftArrow", false)
            break
        case 39: /* right arrow; move right */
            activateBtn("rightArrow", false)
            break
        case 32: /* space dropFast() */
            activateBtn("spacebar", false)
            break
        case 38: /* up arrow; rotate right */
            activateBtn("upArrow", false)
            break
        case 40: /* down arrow; drop piece */
            activateBtn("downArrow", false)
            break
        case 88: /* x; rotate right */
            activateBtn("xBtn", false)
            break
        case 90: /* z; rotate left */
            activateBtn("zBtn", false)
            break
        case 16: /* shift; hold piece */
            activateBtn("shiftBtn", false)
            break
    }
}
function assignPiece(): TetrominoShape {
    const pieces: TetrominoShape[] = ['T', 'J', 'L', 'O', 'S', 'Z', 'I']
    currentPiece.value = nextPiece.value
    nextPiece.value = pieces[pieces.length * Math.random() | 0]
    return nextPiece.value
}
function assignHeldPiece() {
    if (isFirstHold.value === false) {
        initiateNewGamePiece(heldPiece.value)
        heldPiece.value = currentPiece.value
        loadJailcell(heldPiece.value)
    }
    else {
        initiateNewGamePiece(standby.value)
        heldPiece.value = currentPiece.value
        loadJailcell(currentPiece.value)
        loadBullpen()
        assignPiece()
    }

    isFirstHold.value = false
}
function hold() {
    // Don't let players hold twice on the same tetromino
    if (isHeldAlready.value === false) {
        assignHeldPiece()
        isHeldAlready.value = true
    }
}
function canvas(height: number, width: number) {
    const space = []

    while (height--) {
        space.push(new Array(width).fill(0))
    }

    return space
}
function clearRow() {
    let rows = 0

    loop: for (let y = gamearena.length - 1; y > 0; --y) {
        for (let x = 0; x < gamearena[y].length; ++x) {
            if (gamearena[y][x] === 0) {
                continue loop
            }
        }

        let row = gamearena.splice(y, 1)[0].fill(0)
        gamearena.unshift(row)
        ++y

        rows++
        lines.value += 1
    }
    updateScore(rows)
}
function updateScore(rows: number) {
    switch (rows) {
        case 1:
            score.value += 10
            break
        case 2:
            score.value += 30
            break
        case 3:
            score.value += 50
            break
        case 4:
            score.value += 80
            break
    }
    if (lines.value >= 5 * level.value) {
        level.value += 1
        if (dropSpeed.value > 100) {
            dropSpeed.value -= 100
            originalDropSpeed.value = dropSpeed.value
        }
    }
}
function collision() {
    for (let y = 0; y < gamepiece.matrix?.length; ++y) {
        for (let x = 0; x < gamepiece.matrix?.[y].length; ++x) {
            if (gamepiece.matrix?.[y][x] !== 0 && (gamearena[y + gamepiece.position.y] && gamearena[y + gamepiece.position.y][x + gamepiece.position.x]) !== 0) {
                if (dropSpeed.value == 0) { //if dropFast was used
                    dropSpeed.value = originalDropSpeed.value //reset the dropSpeed after tetromino lands
                    document.addEventListener('keydown', playercontrols) //re-enable controls
                }
                return true
            }
        }
    }
    return false
}
function dropShape() {
    if(gameOverStatus.value == false) {
        gamepiece.position.y++

        if (collision()) {
            gamepiece.position.y--
            fuse()
            if (gameOverStatus.value == false) {
                initiateNewGamePiece(standby.value)
                loadBullpen()
            }
            clearRow()
        }

        dropCounter.value = 0
    }
}
function dropFast() {
    document.removeEventListener('keydown', playercontrols) //disable controls until the tetromino lands
    dropSpeed.value = 0

    dropShape()
}
function fuse() {
    gamepiece.matrix?.forEach((row, y) => {
        row.forEach((column, x) => {
            if (column !== 0)
            {
                try {
                    gamearena[y + gamepiece.position.y][x + gamepiece.position.x] = column
                }
                catch {
                    console.log("Boundaries error")
                    gameOver()
                }
            }
        })
    })
    isHeldAlready.value = false //Reset double-hold
}
function pauseToggle() {
    if(pauseStatus.value === false) {
        pauseStatus.value = true
        document.removeEventListener('keydown', playercontrols) //disable controls while paused
        cancelAnimationFrame(cancelId.value)
    }
    else {
        pauseStatus.value = false
        document.addEventListener('keydown', playercontrols) //reenable controls
        requestAnimationFrame(run)
    }
}
function gameOver() {
    document.removeEventListener('keydown', playercontrols) //disable controls (restart key still works)
    document.removeEventListener('keydown', pauseListener) //disable Pause on the letter P
    gameOverStatus.value = true
    document.querySelector("#gameOverMsg").style.display = "block"

    //only prompt to enter name if the leaderboard successfully loaded from API & score is > 0
    if (document.querySelector("#leaderboard")?.style.display == "block" && score.value > 0) {
        document.querySelector("#namePrompt").style.display = "block"
        document.querySelector("#playername").focus()
        document.querySelector("#highscore").value = score.value
    }
}
function restartGame() {
    location.reload();
}
function gamePiece(shape: TetrominoShape): TetrominoMatrix | undefined {
    switch (shape) {
        case 'I': 
            return [
                [0, 0, 0, 0], 
                [1, 1, 1, 1], 
                [0, 0, 0, 0], 
                [0, 0, 0, 0]
            ]
        case 'J': 
            return [
                [2, 0, 0], 
                [2, 2, 2], 
                [0, 0, 0]
            ]
        case 'L': 
            return [
                [0, 0, 3], 
                [3, 3, 3], 
                [0, 0, 0]
            ]
        case 'O': 
            return [
                [4, 4], 
                [4, 4]
            ]
        case 'S': 
            return [
                [0, 5, 5], 
                [5, 5, 0], 
                [0, 0, 0]
            ]
        case 'T': 
            return [
                [0, 0, 0], 
                [6, 6, 6], 
                [0, 6, 0]
            ]
        case 'Z': 
            return [
                [7, 7, 0], 
                [0, 7, 7], 
                [0, 0, 0]
            ]
    }
}
function initiateNewGamePiece(n: TetrominoShape) {
    if (gameOverStatus.value == false) {
        gamepiece.matrix = gamePiece(n)
        gamepiece.position.x = (gamearena[0].length / 2 | 0) - (gamepiece.matrix[0].length / 2 | 0)
        gamepiece.position.y = 0

        if (collision()) {
            gameOver()
        }
    }
}
function loadBullpen() {
    standby.value = assignPiece()

    bullpencontext.value.clearRect(0, 0, bullpencanvas.value?.width, bullpencanvas.value?.height)
    bullpencontext.value.fillStyle = "rgba(255, 255, 255, 0)"
    bullpencontext.value.fillRect(0, 0, bullpencanvas.value?.width, bullpencanvas.value?.height)

    bullpenpiece.matrix = gamePiece(standby.value)

    renderElement(bullpen, { x: 0, y: 1 }, bullpencontext.value)
    renderElement(bullpenpiece.matrix, { x: 0, y: 0 }, bullpencontext.value)
}
function loadJailcell(piece: TetrominoShape) {
    jailcellcontext.value.clearRect(0, 0, jailcellcanvas.value?.width, jailcellcanvas.value?.height)
    jailcellcontext.value.fillStyle = "rgba(255, 255, 255, 0)"
    jailcellcontext.value.fillRect(0, 0, jailcellcanvas.value?.width, jailcellcanvas.value?.height)

    jailcellpiece.matrix = gamePiece(piece)

    renderElement(jailcell, { x: 0, y: 1 }, jailcellcontext.value)
    renderElement(jailcellpiece.matrix, { x: 0, y: 0 }, jailcellcontext.value)
}
function drawCanvases() {
    maincontext.value.clearRect(0, 0, maincanvas.value?.width, maincanvas.value?.height)
    maincontext.value.drawImage(bg.value, 0, 0, 10, 20)

    renderElement(gamearena, { x: 0, y: 0 }, maincontext.value)
    renderElement(gamepiece.matrix, gamepiece.position, maincontext.value)
}
function drawGhosts() {
    renderGhost(ghostarena, { x: 0, y: 0 }, ghostcontext.value)
    renderGhost(gamepiece.matrix, gamepiece.position, maincontext.value)
}
function renderElement(element: HTMLElement, offset: {x: number, y: number}, context: CanvasRenderingContext2D ) {
    element.forEach((row, ypos) => {
        row.forEach((color, xpos) => {
            if (color !== 0) {
                context.globalCompositeOperation='source-over' //puts the Tetromino atop the other elements (like z-index)
                context.drawImage(cube.value, xpos + offset.x, ypos + offset.y, 1, 1)
                context.fillStyle = "rgba(" + colors[color] + ", 0.6)"
                context.fillRect(xpos + offset.x, ypos + offset.y, 1, 1)
            }
        });
    });
}
function renderGhost(element: HTMLElement, offset: {x: number, y: number}, context: CanvasRenderingContext2D) {
    element.forEach((row, ypos) => {
        row.forEach((color, xpos) => {
            if (color !== 0) {
                context.globalCompositeOperation='destination-over' //puts the ghost under the Tetromino (like z-index)
                // if(dropSpeed != 0) {
                    context.fillStyle = "#999"
                    context.fillRect(xpos + offset.x, ypos + offset.y, 1, 20)
                // } else { //add speed trails if dropFast was used
                //     context.fillStyle = "rgba(" + colors[color] + ", 1)"
                //     context.fillRect(xpos + offset.x, ypos + offset.y, 1, -15)
                // }
                
            }
        });
    });
}
function rotate(shape: TetrominoMatrix | undefined, direction: number) {
    for (let y = 0; y < shape.length; ++y) {
        for (let x = 0; x < y; ++x) {
            [shape[x][y], shape[y][x]] = [shape[y][x], shape[x][y]];
        }
    }

    if (direction > 0) {
        shape?.forEach((row) => { 
            row.reverse(); 
        });
    }
    else {
        shape?.reverse();
    }
}
function rotateShape(direction: number) {
    // let offset = 1;

    rotate(gamepiece.matrix, direction);

    while (collision()) {
        rotate(gamepiece.matrix, -direction);

        // gamepiece.position += offset;
        // offset = -(offset + (offset > 0 ? 1 : -1));

        // if (offset > gamepiece.matrix[0].length) {
        //     rotate(gamepiece.matrix, -direction);
        //     gamepiece.position.x = gamepiece.position;
        //     return;
        // }
    }
}
function run(t = 0) {
    const newTime = t - time.value

    dropCounter.value += newTime

    if(dropSpeed.value === 0) {
        dropShape()
        dropShape()
    }

    if (dropCounter.value > dropSpeed.value) {
        dropShape()
    }

    time.value = t

    drawCanvases()
    drawGhosts()
    cancelId.value = requestAnimationFrame(run)
}
function shiftShape(offset: number) {
    gamepiece.position.x += offset;

    if (collision()) {
        gamepiece.position.x -= offset;
    }
}
function initializeFirebase () {
    // Initialize Firebase
    const firebaseApp = initializeApp( {
        apiKey: "AIzaSyBdtpqJx7-NCx643lDKgfAzz5Gz_XNhFmw",
        authDomain: "jstetris-2b64d.firebaseapp.com",
        projectId: "jstetris-2b64d",
        storageBucket: "jstetris-2b64d.appspot.com",
        messagingSenderId: "926826076612",
        appId: "1:926826076612:web:c79d1f239a6a058c236c91"
    });

    // self.FIREBASE_APPCHECK_DEBUG_TOKEN = true;
    // Initialize services
    initializeAppCheck(firebaseApp, {
        provider: new ReCaptchaV3Provider('6Ley3HIhAAAAANlShYkJQQtoCpeZbJLR00-HCVmM')
    });
    const db = getFirestore(firebaseApp);
    const colRef = collection(db, 'highscores');
    // Query db for top highscores
    const leaders = query(colRef, orderBy('highscore', 'desc'), orderBy('timestamp'), limit(100));

    // Set up leaderboard
    const outputTable = document.querySelector("#leaderboard");
    let text = "<h2>LEADERBOARD</h2><table>";
    let xtraDigit = "0";
    let count = 1;

    // Render leaderboard
    function renderLeaderboard(doc) {
        if (count > 9) {
            xtraDigit = "";
        }

        const timestampTooltip = new Date(doc.data().timestamp.seconds*1000).toLocaleDateString("en-US", { year: 'numeric', month: '2-digit', day: '2-digit', hour: '2-digit', minute: '2-digit', second: '2-digit' });

        text += "<tr title='" + timestampTooltip + "'><td style='padding-right: 50px; padding-left: 15px;'>" + xtraDigit.concat(count) + "</td><td style='padding-right: 150px;'>" + doc.data().name + "</td><td style='padding-right: 15px;'>" + doc.data().highscore + "</td></tr>";
        count++;
    }

    // Iterate through the queried db
    getDocs(leaders)
        .then((snapshot) => {
            document.querySelector('#leaderboard').style.display = "block";
            let highscores = []

            snapshot.docs.forEach((doc) => {
                highscores.push({ ...doc.data(), id: doc.id });
                renderLeaderboard(doc);
            })
            text += "</table>";
            outputTable.innerHTML = text;
        })
        .catch (err => {
            console.log(err.message);
        })

    // Add highscore to db
    const addScoreForm = document.querySelector(".addScore")
    addScoreForm.addEventListener('submit', (e) => {
        document.querySelector("#addScoreBtn").disabled = true; //Disable button so you can only submit once

        e.preventDefault()

        addDoc(colRef, {
            name: addScoreForm.name.value.toUpperCase(),
            highscore: parseInt(document.querySelector('#score').innerText),
            timestamp: serverTimestamp()
        })
        .then(() => {
            addScoreForm.reset()
            document.querySelector('#namePrompt').style.display = "none";
            restartGame()
        })
        .catch (err => {
            console.log(err.message);  
        })
    });
}
// #endregion methods

// #region lifecycles
onMounted(() => {
    //Automatically update copyright year
    let year = new Date().getFullYear();
    document.querySelector("#copyrightDate").innerHTML = "&copy;" + year + " Developed by Tommy";

    maincanvas.value = document.querySelector('#gamearena')
    ghostcanvas.value = document.querySelector('#ghostarena')
    bullpencanvas.value = document.querySelector('#bullpenarea')
    jailcellcanvas.value = document.querySelector('#jailcellarea')

    maincontext.value = maincanvas.value?.getContext('2d')
    ghostcontext.value = ghostcanvas.value?.getContext('2d')
    bullpencontext.value = bullpencanvas.value?.getContext('2d')
    jailcellcontext.value = jailcellcanvas.value?.getContext('2d')

    maincontext.value.scale(30, 30)
    ghostcontext.value.scale(30, 30)
    bullpencontext.value.scale(20, 20)
    jailcellcontext.value.scale(20, 20)

    bg.value = document.querySelector('#bg')
    cube.value = document.querySelector('#cube')
    tetris.value = document.querySelector('#tetris');
    maincontext.value.drawImage(bg.value, 0, 0, 10, 20)

    initializeFirebase()
})
onBeforeMount(() => {
    standby.value = assignPiece()

    document.addEventListener('keydown', kbcontrols) //For the "Press ENTER to start"
})
// #endregion lifecycles
</script>

<template>
    <div>
        <div id="assets">
            <img id="bg" src="/src/assets/bg.png" height="400" width="200" />
            <img id="cube" src="/src/assets/cube.png" height="20" width="20" />
        </div>
        <div class="keyboard">
            <div id="leftArrow">&#8592;</div><span id="leftArrowTxt">MOVE<br>LEFT</span>
            <div id="rightArrow">&#8594;</div><span id="rightArrowTxt">MOVE<br>RIGHT</span>
            <div id="upArrow">&#8593;</div><span id="upArrowTxt">ROTATE RIGHT</span>
            <div id="downArrow">&#8595;</div><span id="downArrowTxt">SOFT DROP</span>
            <div id="spacebar">______</div><span id="spaceTxt">HARD DROP</span>
            <div id="zBtn">Z</div><span id="zTxt">ROTATE<br>LEFT</span>
            <div id="xBtn">X</div><span id="xTxt">ROTATE<br>RIGHT</span>
            <div id="shiftBtn">Shift</div><span id="shiftTxt">HOLD</span>
            <div id="escBtn">Esc</div><span id="escTxt">RESTART</span>
            <div id="pBtn" :style="{'background-color': pauseStatus ? '#AAAAFF' : '#DDD'}">P</div>
            <span id="pTxt" style="top: -74px;" :style="{ left: pauseStatus ? '535px' : '547px'}">{{ pauseStatus ? 'UNPAUSE' : 'PAUSE' }}</span>
        </div>
        <div id="tetris">
            <h1>VUETRIS</h1>
            <div id="sidebar-left">
                <label>HOLD</label>
                <br><br>
                <div id="jailcell">
                    <canvas id="jailcellarea" height="80" width="80"></canvas>
                </div>
            </div>
            <div id="main">
                <div id="startDirection"><p>PRESS ENTER<br>TO START</p></div>
                <div id="gameOverMsg"><p>GAME OVER<br>TRY AGAIN</p></div>
                <canvas id="ghostarena" height="600" width="300"></canvas>
                <canvas id="gamearena" height="600" width="300"></canvas>
            </div>
            <div id="sidebar">
                <div id="scoreboard">
                    <div>
                        <p>TIME</p>
                        <span id="timer">{{ timerValue }}</span>
                    </div><hr>
                    <div>
                        <p>SCORE</p>
                        <span id="score">{{ score }}</span>
                    </div><hr>
                    <div>
                        <p>LVL</p>
                        <span id="level">{{ level }}</span>
                    </div><hr>
                    <div>
                        <p>LINES</p>
                        <span id="lines">{{ lines }}</span>
                    </div><hr>
                    <br>
                    <label>NEXT</label>
                    <br><br>
                    <div id="bullpen">
                        <canvas id="bullpenarea" height="80" width="80"></canvas>
                    </div>
                </div>
            </div>
        </div>

        <div id="leaderboard"></div>

        <div id="namePrompt">
            <form class="addScore">
                <label for="playerName">Name:</label>
                <input id="playername" type="text" name="name" minlength="3" maxlength="3" placeholder="AAA" onkeypress="return (event.keyCode >= 97 && event.keyCode <= 122) || (event.keyCode >= 65 && event.keyCode <= 90)" autocomplete="off" required><br>
                <label for="highScore">Score:</label>
                <input id="highscore" type="number" name="highscore" readonly="readonly">

                <input id="timestamp" type="datetime" name="timestamp" readonly="readonly"> <!--This is invisible. For backend purposes only-->

                <br>
                <button id="addScoreBtn">OK</button>
            </form>
        </div>
        
        <footer>
            <p id="copyrightDate"></p>
        </footer>
    </div>
</template>

<style lang='scss'>
@font-face {
    font-family: 'Press_Start';
    src: url('/src/fonts/PressStartRegular.woff2') format('woff2'),
         url('/src/fonts/PressStartRegular.woff') format('woff');
    font-weight: normal;
    font-style: normal;
}

html {
    overflow: hidden; /*Prevent scrolling via the arrow keys*/
}
body {
    font-family: "Press_Start";
}

#assets
{
    display: none;
}

.keyboard {
    font-size: 12px;
    position: absolute;
    left: 40px;
    top: 650px;
}
.keyboard div {
    background-color: #DDD;
    width: 30px;
    height: 25px;
    position: absolute;
    border: 1px gray solid;
    text-align: center;
    padding-top: 5px;
    font-family: Arial, sans-serif;
}
#leftArrow {
    left: 530px;
    top: 60px;
}
#rightArrow {
    left: 590px;
    top: 60px;
}
#downArrow {
    left: 560px;
    top: 60px;
}
#upArrow {
    left: 560px;
    top: 30px;
}
#spacebar {
    width: 180px;
    left: 240px;
    top: 60px;
}
#zBtn {
    left: 55px;
    top: 60px;
}
#xBtn {
    left: 85px;
    top: 60px;
}
#shiftBtn {
    left: -17px;
    top: 111px;
    width: 75px;
}
#escBtn {
    left: 105px;
    top: 110px;
    width: 45px;
}
#pBtn {
    left: 560px;
    top: -60px;
}
.keyboard span {
    position: absolute;
    text-align: center;
}
#leftArrowTxt {
    left: 480px;
    top: 63px;
}
#rightArrowTxt {
    left: 626px;
    top: 63px;
}
#downArrowTxt {
    left: 554px;
    top: 96px;
}
#upArrowTxt {
    left: 544px;
    top: 1px;
}
#spaceTxt {
    left: 277px;
    top: 97px;
    width: 108px;
}
#zTxt {
    left: -21px;
    top: 63px;
}
#xTxt {
    left: 121px;
    top: 63px;
}
#shiftTxt {
    left: px;
    top: 147px;
}
#escTxt {
    left: 87px;
    top: 147px;
}
#pTxt {
    left: 547px;
    top: -74px;
}

h1 {
    position: absolute;
    top: 5px;
    left: 165px;
    color: #3f8657;
}

#tetris {
    background-color: #AAA;
    background-image: linear-gradient(#e1ede3, #a9d4b6);
    border: solid 2px #3f8657;
    border-radius: 16px;
    display: flex;
    height: 630px;
    padding: 1.5rem;
    width: 510px;
}

#ghostArena {
    position: fixed;
}

#main
{
    position: absolute;
    top: 70px;
    left: 120px;

    &#ghostarena, #gamearena {
    position: absolute;
    top: 0;
    left: 0;
}
}

#startDirection {
    text-align: center;
    line-height: 1.5rem;
    display: block;
    position: absolute;
    z-index: 999;
    top: 250px;
    left: 52px;
    color: #fff;
    font-size: 1rem;
    background-color: rgba(0,0,0,.6);
    padding: 0 10px;
}

#gameOverMsg {
    display: none;
    position: absolute;
    z-index: 999;
    top: 250px;
    left: 69px;
    color: #fff;
    font-size: 1rem;
    background-color: rgba(0,0,0,.6);
    padding: 0 10px;
    text-align: center;
    line-height: 1.5rem;
}

hr {
    border: 1px solid #3f8657;
}
#sidebar-left
{
    position: absolute;
    color: #000;
    padding-left: 20px;
    width: 100px;
    font-weight: bold;
    top: 75px;
    left: 10px;
}
#sidebar
{
    position: absolute;
    color: #000;
    padding-left: 20px;
    width: 100px;
    font-weight: bold;
    top: 55px;
    left: 420px;
}

#sidebar div {
    padding-bottom: 10px;
}
#sidebar div p {
    margin-bottom: 5px;
}

a {
    color: #000;
    font-weight: bold;
}

#leaderboard {
    position: absolute;
    top: 8px;
    left: 700px;
    padding: 16px 30px;
    background-color: #AAA;
    background-image: linear-gradient(#e1ede3, #a9d4b6);
    border: solid 2px #3f8657;
    border-radius: 16px;
    display: none;
    overflow-y: scroll;
    max-height: 500px;
}
#leaderboard table {
    border-collapse: collapse;
    margin-left: -15px;
    margin-right: -15px;
}
#leaderboard td {
    text-align: left;
    font-weight: normal;
    padding-bottom: 8px;
    padding-top: 8px;
    border-collapse: collapse;
}
#leaderboard h2 {
    font-size: 20px;
    color: #3f8657;
    text-align: center;
}
tr:hover {
    background-color: #00000020;
}

::-webkit-scrollbar {
    width: 5px;
}
::-webkit-scrollbar-track {
    background: transparent;
    margin: 30px 0;
}
::-webkit-scrollbar-thumb {
    background: #3f8657;
    border-radius: 5px;
}

#namePrompt {
    display: none;
    position: absolute;
    top: 285px;
    left: 600px;
    background-color: #AAA;
    background-image: linear-gradient(#e1ede3, #a9d4b6);
    border: solid 2px #3f8657;
    border-radius: 16px;
    padding: 3rem;
}
#playername {
    font-family: "Press_Start";
    font-size: 20px;
    padding: 5px 0 5px 5px;
    background-color: black;
    color: #00CC00;
    border: none;
}
#highscore {
    margin-top: 2rem;
    border: none;
    background-color: transparent;
    color: black;
    font-family: "Press_Start";
    font-size: 20px;
}
#highscore:focus {
    outline: none;
}
#timestamp {
    display: none;
}
#addScoreBtn {
    margin-top: 2rem;
    font-family: "Press_Start";
    background-color: #00CC00;
    padding: 1rem 3rem;
    color: white;
    border: black 2px solid;
}
#addScoreBtn:hover {
    cursor: pointer;
}

footer {
    position: absolute;
    bottom: 0;
    width: 100%;
    text-align: center;
    font-family: Arial, sans-serif;
}
</style>