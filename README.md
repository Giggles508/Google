<!DOCTYPE html>
<html lang="de">
<head>
<meta charset="UTF-8">
<title>NEON BLACKJACK</title>
<link href="https://fonts.googleapis.com/css2?family=Orbitron:wght@700&display=swap" rel="stylesheet">

<style>
body{
    margin:0;
    background:black;
    font-family:'Orbitron',sans-serif;
    color:#0ff;
    text-align:center;
}

h1{
    text-shadow:0 0 20px #0ff;
}

.box{
    border:2px solid #0ff;
    box-shadow:0 0 25px #0ff;
    border-radius:20px;
    padding:20px;
    width:420px;
    margin:20px auto;
}

button{
    padding:10px 15px;
    margin:5px;
    border-radius:10px;
    border:2px solid #ff00ff;
    background:black;
    color:#ff00ff;
    cursor:pointer;
    font-weight:bold;
}
button:hover{
    background:#ff00ff;
    color:black;
    box-shadow:0 0 20px #ff00ff;
}

input{
    padding:8px;
    border-radius:10px;
    border:2px solid #0ff;
    background:black;
    color:#0ff;
    text-align:center;
}

.card{
    display:inline-block;
    width:60px;
    height:90px;
    background:white;
    color:black;
    border-radius:10px;
    line-height:90px;
    margin:5px;
    font-weight:bold;
    font-size:18px;
    animation:deal .3s ease;
}

@keyframes deal{
    from{transform:translateY(-50px);opacity:0;}
    to{transform:translateY(0);opacity:1;}
}
</style>
</head>
<body>

<h1>⚡ NEON BLACKJACK ⚡</h1>

<div id="loginBox" class="box">
    <h3>Dein Name</h3>
    <input type="text" id="username">
    <br><br>
    <button onclick="login()">START</button>
</div>

<div id="gameBox" class="box" style="display:none;">
    <h3>Spieler: <span id="playerName"></span></h3>
    <h3>Geld: $<span id="balance">100</span></h3>
    Einsatz:
    <input type="number" id="bet" value="10" min="1">
    <br><br>

    <button onclick="startGame()">Neue Runde</button>

    <h3>Dealer (<span id="dealerScore">0</span>)</h3>
    <div id="dealerCards"></div>

    <h3>Du (<span id="playerScore">0</span>)</h3>
    <div id="playerCards"></div>

    <div id="controls"></div>
    <h2 id="result"></h2>

    <div id="riskBox" style="display:none;">
        <h3>⚡ Gewinn verdoppeln?</h3>
        <button onclick="risk(true)">JA</button>
        <button onclick="risk(false)">NEIN</button>
    </div>

    <p>Spiele: <span id="games">0</span> | Siege: <span id="wins">0</span></p>

    <button onclick="resetGame()">Reset</button>
</div>

<audio id="winSound" src="https://assets.mixkit.co/sfx/preview/mixkit-winning-chimes-2015.mp3"></audio>
<audio id="loseSound" src="https://assets.mixkit.co/sfx/preview/mixkit-arcade-retro-game-over-213.mp3"></audio>

<script>
let balance=100;
let playerName="";
let deck=[];
let player=[];
let dealer=[];
let games=0;
let wins=0;
let lastWin=0;

function login(){
    let name=document.getElementById("username").value.trim();
    if(!name)return alert("Bitte Namen eingeben");

    playerName=name;

    let saved=JSON.parse(localStorage.getItem("blackjackData"));
    if(saved){
        balance=saved.balance||100;
        games=saved.games||0;
        wins=saved.wins||0;
    }

    document.getElementById("loginBox").style.display="none";
    document.getElementById("gameBox").style.display="block";
    updateUI();
}

function save(){
    localStorage.setItem("blackjackData",JSON.stringify({
        balance,games,wins
    }));
}

function updateUI(){
    document.getElementById("playerName").innerText=playerName;
    document.getElementById("balance").innerText=balance;
    document.getElementById("games").innerText=games;
    document.getElementById("wins").innerText=wins;
    save();
}

function createDeck(){
    let suits=["♠","♥","♦","♣"];
    let values=["A","2","3","4","5","6","7","8","9","10","J","Q","K"];
    deck=[];
    for(let s of suits){
        for(let v of values){
            deck.push({value:v,suit:s});
        }
    }
}

function shuffle(){
    for(let i=deck.length-1;i>0;i--){
        let j=Math.floor(Math.random()*(i+1));
        [deck[i],deck[j]]=[deck[j],deck[i]];
    }
}

function getCardValue(card){
    if(card.value==="A") return 11;
    if(["J","Q","K"].includes(card.value)) return 10;
    return parseInt(card.value);
}

function calculateScore(hand){
    let score=0;
    let aces=0;

    for(let c of hand){
        score+=getCardValue(c);
        if(c.value==="A") aces++;
    }

    while(score>21 && aces>0){
        score-=10;
        aces--;
    }

    return score;
}

function render(){
    document.getElementById("playerCards").innerHTML="";
    document.getElementById("dealerCards").innerHTML="";

    player.forEach(c=>{
        document.getElementById("playerCards").innerHTML+=
        `<div class="card">${c.value}${c.suit}</div>`;
    });

    dealer.forEach(c=>{
        document.getElementById("dealerCards").innerHTML+=
        `<div class="card">${c.value}${c.suit}</div>`;
    });

    document.getElementById("playerScore").innerText=calculateScore(player);
    document.getElementById("dealerScore").innerText=calculateScore(dealer);
}

function startGame(){
    let bet=parseInt(document.getElementById("bet").value);
    if(bet>balance)return alert("Nicht genug Geld");

    createDeck();
    shuffle();

    player=[deck.pop(),deck.pop()];
    dealer=[deck.pop(),deck.pop()];

    document.getElementById("controls").innerHTML=
    `<button onclick="hit()">Hit</button>
     <button onclick="stand()">Stand</button>`;

    document.getElementById("result").innerText="";
    document.getElementById("riskBox").style.display="none";

    render();
}

function hit(){
    player.push(deck.pop());
    render();
    if(calculateScore(player)>21){
        endGame(false);
    }
}

function stand(){
    while(calculateScore(dealer)<17){
        dealer.push(deck.pop());
    }
    render();

    let p=calculateScore(player);
    let d=calculateScore(dealer);

    if(d>21 || p>d){
        endGame(true);
    }else if(p===d){
        document.getElementById("result").innerText="Unentschieden";
    }else{
        endGame(false);
    }
}

function endGame(win){
    let bet=parseInt(document.getElementById("bet").value);
    games++;

    if(win){
        balance+=bet;
        wins++;
        lastWin=bet;
        document.getElementById("result").innerText="🎉 Gewonnen!";
        document.getElementById("winSound").play();
        document.getElementById("riskBox").style.display="block";
    }else{
        balance-=bet;
        document.getElementById("result").innerText="💀 Verloren!";
        document.getElementById("loseSound").play();
    }

    document.getElementById("controls").innerHTML="";
    updateUI();
}

function risk(choice){
    if(!choice){
        document.getElementById("riskBox").style.display="none";
        return;
    }

    if(Math.random()>0.5){
        lastWin*=2;
        balance+=lastWin;
        document.getElementById("result").innerText="⚡ Verdoppelt!";
        document.getElementById("winSound").play();
    }else{
        balance-=lastWin;
        document.getElementById("result").innerText="💥 Risiko verloren!";
        document.getElementById("loseSound").play();
        document.getElementById("riskBox").style.display="none";
    }

    updateUI();
}

function resetGame(){
    localStorage.removeItem("blackjackData");
    location.reload();
}
</script>

</body>
</html>
