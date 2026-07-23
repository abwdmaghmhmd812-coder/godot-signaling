// سيرفر Signaling بسيط - ينقل رسائل التعارف بين اللاعبين فقط
// ما يشغل اللعبة، بس يساعد اللاعبين "يلاقوا بعض"

const WebSocket = require('ws');
const http = require('http');

const PORT = process.env.PORT || 8080;

const server = http.createServer();
const wss = new WebSocket.Server({ server });

// كل غرفة (Lobby) فيها لاعبين، كل واحد له id
let rooms = {}; // { roomCode: { peerId: ws } }

function generateRoomCode() {
	return Math.floor(1000 + Math.random() * 9000).toString(); // كود من 4 أرقام
}

wss.on('connection', (ws) => {
	let currentRoom = null;
	let peerId = null;

	ws.on('message', (message) => {
		let data;
		try {
			data = JSON.parse(message);
		} catch (e) {
			return;
		}

		// إنشاء غرفة جديدة (Host)
		if (data.type === 'create_room') {
			const roomCode = generateRoomCode();
			rooms[roomCode] = {};
			peerId = 1; // الهوست دايماً ID = 1
			rooms[roomCode][peerId] = ws;
			currentRoom = roomCode;

			ws.send(JSON.stringify({
				type: 'room_created',
				room_code: roomCode,
				peer_id: peerId
			}));
			console.log(`Room created: ${roomCode}`);
		}

		// الانضمام لغرفة موجودة (Join)
		else if (data.type === 'join_room') {
			const roomCode = data.room_code;
			if (!rooms[roomCode]) {
				ws.send(JSON.stringify({ type: 'error', message: 'الغرفة غير موجودة' }));
				return;
			}

			const existingIds = Object.keys(rooms[roomCode]).map(Number);
			peerId = Math.max(...existingIds) + 1;
			rooms[roomCode][peerId] = ws;
			currentRoom = roomCode;

			// خبر اللاعب الجديد بمعرفه ومعرفات الموجودين
			ws.send(JSON.stringify({
				type: 'room_joined',
				peer_id: peerId,
				existing_peers: existingIds
			}));

			// خبر باقي اللاعبين بوجود لاعب جديد
			for (const id of existingIds) {
				rooms[roomCode][id].send(JSON.stringify({
					type: 'new_peer',
					peer_id: peerId
				}));
			}
			console.log(`Peer ${peerId} joined room ${roomCode}`);
		}

		// تمرير رسائل WebRTC (offer/answer/ice candidate) بين اللاعبين
		else if (data.type === 'signal') {
			const targetId = data.target_id;
			if (currentRoom && rooms[currentRoom] && rooms[currentRoom][targetId]) {
				rooms[currentRoom][targetId].send(JSON.stringify({
					type: 'signal',
					from_id: peerId,
					payload: data.payload
				}));
			}
		}
	});

	ws.on('close', () => {
		if (currentRoom && rooms[currentRoom]) {
			delete rooms[currentRoom][peerId];
			// خبر الباقين إنه فصل
			for (const id in rooms[currentRoom]) {
				rooms[currentRoom][id].send(JSON.stringify({
					type: 'peer_left',
					peer_id: peerId
				}));
			}
			if (Object.keys(rooms[currentRoom]).length === 0) {
				delete rooms[currentRoom];
			}
		}
	});
});

server.listen(PORT, () => {
	console.log(`Signaling server running on port ${PORT}`);
});
