-----------------------------Front-----------------------------
layout.html{
	user = req.user;
	
	if(user) {
		로그인 { post('/auth/login), email, password }, 회원가입{ get('join') } 
	} else {
		로그아웃 { get('/auth/logout') }, 낙찰 내역 { get('/list') }, 상품 등록 { get('/good') }
	}
}

main.html{
	goods = Good.findAll(where:{ OwnerId: null });
	
	for good in goods
		good.name good.img, good.price, td { sse 시간 } , 입장 { get('/good/{{good.id}}') }
	
	<script>
	es = EventSource('/see');
	es.onmessage = function (e) { 
		end = new Date(td.dataset.start); // 경매 시작 시간
		server = new Date(parseInt(e.data, 10)); // 서버센트에서 보낸 시간
		end.setDate(end.getData() + 1); //경매 종료 시간
		return td.textContent = 남은 시간
	}
	</script>
	
	//서버 시간으로 굳이 하는 이유 = 클라이언트 못믿음, 다 같은 시각에 시작, 종료되어야 하기에 1개로 통일 필요
}

join.html{
	회원가입 { post('/auth/join'), email, nick, password, join-money }
}

good.html{
	상품 등록 { post('/good'), name, img, price }
}

auction.html{
	block good {
		good = Good.findOne(where: { id: req.params.id });
		good.name
		good.Owner.nick
		good.price원
		time(data-start={{good.createdAt}}) { sse 시간 }
		"/img/good.img"
		<script>
		es = new EventSource("/sse");
		es.onmessage = (e) => {
			...
			return time.textContent = 남은 시간
		}
		</script>
	}
	
	block content {
		auction = 
		for bid in auction
			bid.User.nick님
			bid.bid원에 입찰하셨습니다.
			bid.msg

		
		입찰 button { bid, msg } //form에 action이 없음
		
		<script>
		document.querySelector('입찰').addEventListener('submit', (e) => {
			axios.post('/good/{{good.id}}/bid') {
				bid: e.target.bid.value,
				msg: e.target.msg.value,
			}
			 .catch((err) => {})
			 .finally(() => {
			 	e.target.bid.value = '';
				e.target.msg.value = '';
				errorMessage.textContent = '';
			 });
		});
		
		scoket = io.connect('http://localhost:8010, {
			path: '/socket.io'
		});
		socket.on('bid, (data) => //입찰했을 때) bid에 추가
	}
}

list.html{
	goods = good.findAll(where: { OwnderId: req.user.id });
	for good in goods
	good.name
	"/img/{{good.img}}"
	{{good.Auctions[0].bid}} //최종 가격
}
-----------------------------Router-----------------------------
index {
	use( res.locals.user = req.user )

	get('/') {
	
	}

	get('/join'){
	
	}

	get('/good'){
	
	}

	post('/good'){
	
	}
	
	get('/good/:id'){
	
	}
	
	post('/good/:id/bid'){
	
	}
	
	get('/list'){
	
	}
}

auth {
	post('/join') {
		{ email, nick, password, money } = req.body;
		exUser = User.findOne(where { email });
		if(exUser) return res.redircet('이미 가입');
		await User.create({
			email, nick, password: hash, moeny
		})
		return res.redirect('/');
	}
	
	post('/login') {
		passport.authenticate('local' => req.login(user));
	}
	
	get('logout') {
		req.logout();
		req.session.destroy();
		res.redirect('/');
	}
}
-----------------------------DB-----------------------------
User

Good

Auction { User N:N Good}

Ownser {User 1:N Good}
Solder {User 1:N Good}

-----------------------------app.js-----------------------------

-----------------------------socket.IO-----------------------------
const SocketIO = require('socket.io');

module.exports = (server, app) => {
  const io = SocketIO(server, { path: '/socket.io' });
  app.set('io', io);
  io.on('connection', (socket) => { // 웹 소켓 연결 시
    const req = socket.request;
    const { headers: { referer } } = req;
    const roomId = referer.split('/')[referer.split('/').length - 1];
    socket.join(roomId);
    socket.on('disconnect', () => {
      socket.leave(roomId);
    });
  });
};

-----------------------------sse-----------------------------
const SSE = require('sse');

module.exports = (server) => {
  const sse = new SSE(server);
  sse.on('connection', (client) => { // 서버센트이벤트 연결
    setInterval(() => {
      client.send(Date.now().toString());
    }, 1000);
  });
};

-----------------------------checkAuction-----------------------------
const { Op } = require('Sequelize');

const { Good, Auction, User, sequelize } = require('./models');

module.exports = async () => {
  console.log('checkAuction');
  try {
    const yesterday = new Date();
    yesterday.setDate(yesterday.getDate() - 1); // 어제 시간
    const targets = await Good.findAll({
      where: {
        SoldId: null,
        createdAt: { [Op.lte]: yesterday },
      },
    });
    targets.forEach(async (target) => {
      const success = await Auction.findOne({
        where: { GoodId: target.id },
        order: [['bid', 'DESC']],
      });
      await Good.update({ SoldId: success.UserId }, { where: { id: target.id } });
      await User.update({
        money: sequelize.literal(`money - ${success.bid}`),
      }, {
        where: { id: success.UserId },
      });
    });
  } catch (error) {
    console.error(error);
  }
};