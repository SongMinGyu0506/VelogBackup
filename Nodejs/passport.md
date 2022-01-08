# Passport 모듈
로그인 및 회원가입, SNS 로그인 등을 직접 구현하지 않고 사용함

---

## passport 설치
```bash
npm i passport passport-local passport-kakao bcrypt
```
* passport-local : 로컬 로그인 모듈
* passport-kakao : kakao SNS 로그인 모듈
* bcrypt : 비밀번호 해싱
---
## passport 설정
### passport 기본 연결
```javascript
//app.js
const dotenv = require('dotenv'); // 설정파일
const passport = require('passport');

dotenv.config();
const pageRouter = require('./routes/page');
const {sequelize} = require('./models');
const passportConfig = require('./passport'); //Passport 설정 import

const app = express();
passportConfig(); //호출
app.set('port',process.env.PORT || 8001);
app.set('view engine','html');
...
//로그인에 사용할 세션 생성
app.use(session({
    resave: false,
    saveUninitialized: false,
    secret: process.env.COOKIE_SECRET,
    cookie: {
        httpOnly: true,
        secure: false,
    },
}));

//req 객체에 passport 설정을 삽입
app.use(passport.initalize());

//req.session 객체에  passport 정보를 저장
// req.session 객체는 express-session 미들웨어를 이용하여 생성하므로 해당 작업은 세션이 생성되어있음을 전제로 진행해야함
app.use(passport.session());

app.use('/',pageRouter);
...
```
### passport 기본 설정 세팅
```javascript
//passport/index.js
const passport = require('passport');
const local = require('./localStrategy');
const kakao = require('./kakaoStrategy');
const User = require('../models/user');

module.exports = () => {
    /*
        serializeUser : 
        로그인 시 실행, req.session(세션) 객체에 어떤 데이터를 저장할지 정하는 메서드
        매개변수로 user, 사용자 정보를 받고 done 함수에 user.id를 넘긴다.

        done의 첫 번째 파라미터는 에러 발생 시 사용, 
        두 번째 인수에는 저장하고 싶은 데이터를 넣는다.
        로그인 시 사용자 데이터를 세션에 저장,

    */
    passport.serializeUser((user,done)=> {
        done(null,user.id);
    });

    /*
        deserializeUser :
        매 요청시 실행되는 메서드
        passport.session 미들웨어가 이 메서드를 호출, serializeUser의 done 두번 째 인수로 넣었던 데이터가 deserializeUser의 매개변수
    */
    passport.deserializeUser((id,done)=> {
        User.findOne({where:{id}})
            .then(user=>done(null,user/*req.user에 저장*/))
            .catch(err=>done(err));
    });

    //로그인 전략
    local();
    kakao();
}
```
---
## passport 대략적인 전체 과정
1. 라우터를 통해 로그인 요청이 들어옴
2. 라우터에서 passport.authenticate 메서드 호출
3. 각 로그인 전략 수행
4. 로그인 성공 시 사용자 정보 객체와 함께 req.login 호출
5. req.login 메서드가 passport.serializeUser 호출
6. req.session에 사용자 아이디만 저장
7. 로그인 완료

### __passport 로그인 이후의 과정__
1. 요청이 들어옴
2. 라우터에 요청이 도달하기 전에 passport.session 미들웨어가 passport.deserializeUser 메서드 호출
3. req.session에 저장된 아이디로 데이터베이스에서 사용자 조회
4. 조회된 사용자 정보를 req.user에 저장
5. 라우터에서 req.user 객체 사용 가능

# Local 로그인 구현
__예제__ 로그인 여부 판단 미들웨어
```javascript
//routes/middlewares.js

/*
    isAuthenticated() :
    인증 처리가 되어있는지 아닌지 확인하는 메서드.
*/
exports.isLoggedIn = (req,res,next) => {
    if (req.isAuthenticated()) {
        next();
    } else {
        res.status(403).send('로그인 필요');
    }
};
exports.isNotLoggedIn = (req,res,next) => {
    if(!req.isAuthenticated()) {
        next();
    } else {
        res.redirect(`/?error=${message}`);
    }
} 
```
