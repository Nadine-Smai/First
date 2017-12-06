# First
gameMode = false;
var $sel = function (c) { return document.querySelector(c); };
var $$sel = function (doc, c) { return doc.querySelectorAll(c); };
var distance = function (e, b) { return Math.sqrt((e.x + e.size.x / 2 - b.x) * (e.x + e.size.x / 2 - b.x) + (e.y - e.size.y / 2 - b.y) * (e.y - e.size.y / 2 - b.y)); };
var clamp = function (v, a, b) { return Math.min(Math.max(v, a), b); };
var drawCircle = function (ctx, x, y, r, func) {
	ctx.beginPath();
	ctx.arc(x, y, r, 0, 2 * Math.PI, false);
	func.call(ctx);
};
var canvas = document.createElement("canvas");
canvas.width = window.innerWidth;
canvas.height = window.innerHeight;
var ctx = canvas.getContext('2d');
var list = null;
var pageCount = 0, linkCount = 0;
function createTagList (doc) {
	if (doc == null) return;
	pageCount++;
	list = Array.prototype.map.call($$sel(doc, 'a'), function (e) { return {
		text: e.innerText,
		href: e.href,
		x: Math.random() * 1000,
		y: Math.random() * 1000,
		size: { x: ctx.measureText(e.innerText).width, y: 10}
	}; });
}
createTagList(document);
var keyCode = { LEFT: 37, RIGHT: 39, UP: 38, DOWN: 40 }, input = {};
window.onkeydown = function (e) {
	e.preventDefault();
	for (var keyName in keyCode)
		if (e.keyCode == keyCode[keyName]) input[keyName] = true;
};
window.onkeyup = function (e) {
	e.preventDefault();
	for (var keyName in keyCode)
		if (e.keyCode == keyCode[keyName]) input[keyName] = false;
};
$sel('body').parentNode.removeChild($sel('body'));
$sel('html').appendChild(document.createElement('body')).appendChild(canvas);
var player = { x: canvas.width / 2, y: canvas.height / 2, velocity: { x: 0, y: 0 } };
var energy = 100.0, invincibleTimer = 25;
var gameUpdateVar;
gameUpdateVar = window.setInterval(function () {
	if (invincibleTimer > 0) invincibleTimer -= 1;
	ctx.fillStyle = 'black';
	ctx.fillRect(0, 0, canvas.width, canvas.height);
	list.forEach(function (e) {
		var alpha = 1 - clamp(distance(e, player) / 100.0 - 1, 0, 1);
		if (e.spent) ctx.fillStyle = 'rgba(0, 255, 255, ' + alpha + ')';
		else ctx.fillStyle = 'rgba(255, 255, 255, ' + alpha + ')';
		if (alpha > 0 && !e.seen) { e.seen = true; linkCount++; }
		ctx.fillText(e.text, e.x, e.y);
		ctx.strokeStyle = 'rgba(' + (e.spent? '0' : '255') + ', 255, 255, ' + alpha / 3 + ')';
		drawCircle(ctx, e.x + e.size.x / 2, e.y - e.size.y / 2, 30, ctx.stroke);
		e.x += Math.random() * 2 - 1;
		e.y += Math.random() * 2 - 1;
	});
	if (input.RIGHT) player.velocity.x = clamp(player.velocity.x + 0.3, -5, 5);
	if (input.LEFT) player.velocity.x = clamp(player.velocity.x - 0.3, -5, 5);
	if (input.UP) player.velocity.y = clamp(player.velocity.y - 0.3, -5, 5);
	if (input.DOWN) player.velocity.y = clamp(player.velocity.y + 0.3, -5, 5);
	player.x = clamp(player.x + player.velocity.x, 0, canvas.width);
	player.y = clamp(player.y + player.velocity.y, 0, canvas.height);
	ctx.fillStyle = 'white';
	drawCircle(ctx, player.x, player.y, 3, ctx.fill);
	if (gameMode) {
		ctx.fillStyle = 'teal';
		ctx.fillRect(0, 0, canvas.width * energy / 100, 10);
	}
	list.forEach(function (e) {
		if (!e.spent && distance(e, player) < 30) { e.spent = true; energy = Math.min(100, energy + 3);}
		if (invincibleTimer == 0 && !e.unavailable && hitTest(player, e)) {
			createTagList(getNewPage(e.href, e));
			invincibleTimer = 25;
			energy -= 15;
		}
	});
	if (energy > 0) energy -= 0.2;
	if (gameMode && energy <= 0) {
		window.clearInterval(gameUpdateVar);
		window.setInterval (function () {
			ctx.fillStyle = 'black';
			ctx.fillRect(0, 0, canvas.width, canvas.height);
			ctx.fillStyle = 'white';
			ctx.textAlign = "center";
			ctx.fillText("Game Over", canvas.width / 2, canvas.height / 2);
			ctx.fillText("You've crawled " + pageCount + " pages and seen " + linkCount + " links.", canvas.width / 2, canvas.height / 2 + 30);
		}, 30);
	}
}, 30);
function hitTest (pos, e) { return pos.x > e.x && pos.y < e.y && pos.x < e.x + e.size.x && pos.y > e.y - e.size.y; }
function getNewPage (url, e) {
	var xmlHttp = new XMLHttpRequest();
	xmlHttp.open("GET", url, false);
	xmlHttp.send(null);
	if (xmlHttp.responseText == null) { e.unavailable = true; return null; }
	return new DOMParser().parseFromString(xmlHttp.responseText, "text/html");
