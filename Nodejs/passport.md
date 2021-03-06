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
## __예제__ 로그인 여부 판단 미들웨어
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
라우팅 렌더링 전, 해당 메서드를 사용하면 로그인이 필요하거나 필요하지 않은 라우팅으로 대체가능

## 관련 라우터 예제 (join,login,logout)
```javascript
...
//회원가입 라우팅
router.post('/join',isNotLoggedIn, async (req,res,next)=>{
    const {email,nick,password} = req.body; // 프론트의 form 데이터
    try {
        const exUser = await User.findOne({where:{email}}); //email로 유저데이터 서치
        
        //이미 존재하는 이메일의 경우,
        if(exUser) {
            return res.redirect('/join?error=exist');
        }

        //비밀번호 해싱처리 bcrypt.hash(password,hasingLength)
        const hash = await bcrypt.hash(password,12);

        //db 등록
        await User.create({
            email,
            nick,
            password:hash,
        });
        return res.redirect('/');

        //error 처리
    } catch (error) {
        console.error(error);
        return next(error)
    }
});

//로그인 시도 라우팅
router.post('/login',isNotLoggedIn, (req,res,next)=>{
    //passport.authenticate('local') 일 경우, 로컬 전략 실행 해당 메서드로 전략 성공 및 실패 여부 판정
    passport.authenticate('local',(authError,user,info)=> {

        //첫번째 파라미터가 존재한다면 해당 전략은 실패
        if (authError) {
            console.error(authError);
            return next(authError);
        }

        //user 데이터가 없어도 실패, 전략에서 지정한 info.message 출력
        if (!user) {
            return res.redirect(`/?loginError=${info.message}`);
        }

        //전략 성공시 req에 login과 logout 메서드를 추가함
        //req.login은 passport.serializeUser를 호출
        return req.login(user,(loginError)=> {
            if(loginError) {
                console.error(loginError);
                return next(loginError);
            }
            return res.redirect('/');
        });
    })(req,res,next);
});

//로그아웃 메서드
router.get('/logout',isLoggedIn,(req,res)=>{
    req.logout();
    req.session.destroy();
    res.redirect('/');
});

module.exports = router;
```
## 로컬 전략 모듈
```javascript
//passport/localStrategy.js
const passport = require('passport');
const LocalStrategy = require('passport-local');
const bcrypt = require('bcrypt');

const User = require('../models/user');

module.exports = () => {
    //usernameFiield와 passwordField는 req.body의 form name
    passport.use(new LocalStrategy({
        usernameField: 'email',
        passwordField:'password',
        //아래 콜백함수부터 전략 처리
    }, async (email,password,done)=>{
        try {
            const exUser = await User.findOne({where:{email}}); //DB에서 해당 유저의 계정으로 유저 데이터 추출
            // 존재하는 계정이라면
            if (exUser) {
                const result = await bcrypt.compare(password,exUser.password); //패스워드가 동일한지 판별 t/f
                if (result) {
                    //정상이라면
                    done(null, exUser); //Error : null, user : exUser, info : null
                } else {
                    //비밀번호 다를경우
                    done(null, false, {message: '비밀번호가 일치하지 않습니다.'}) //에러 x 유저데이터 x, 메시지 송출
                }
            } else {
                done (null, false, {message: '가입되지 않은 회원입니다.'});
            }
        } catch (error) {
            console.error(error);
            done(error);
        }
    }));
};
```

# Kakao SNS 로그인 구현
SNS 로그인의 특징으로 회원가입 절차가 없다.  
첫 로그인에 회원가입 처리 그 이후 부터 로그인 처리하는 전략을 사용해야함

```javascript
const passport = require('passport');
const KakaoStrategy = require('passport-kakao').Strategy;

const User = require('../models/user');

module.exports = () => {
    passport.use(new KakaoStrategy({
        clientID: process.env.KAKAO_ID // 카카오 인증키
        callbackURL: '/auth/kakao/callback', //카카오로부터 인증 결과를 받을 라우터 주소

        //KakaoStrategy 전략 수행 후 accessToken, refreshToken, profile 생성
    }, async (accessToken,refreshToken, profile, done)=> {
        console.log('kakao profile',profile);
        try {
            //카카오 계정 조회
            const exUser = await User.findOne({
                where: {snsId: profile.id, provider:'kakao'}
            });

            //존재하는 유저 데이터일경우 바로 실행
            if (exUser) {
                done(null,exUser);

                //새로운 유저일 경우, 디비에 생성
            } else {
                const newUser = await User.create({
                    email: profile._json&& profile._json.kakao_account_email, //프로필 기준
                    nick: profile.displayName,
                    snsId: profile.id,
                    provider: 'kakao',
                });
                done(null, newUser);
            }
        } catch (error) {
            console.error(error);
            done(error);
        }
    }))
}
```
__routes/auth.js__
```javascript
...
router.get('/kakao',passport.authenticate('kakao'));
router.get('/kakao/callback',passport.authenticate('kakao',{failureRedirect:'/',}),(req,res)=>{
    res.redirect('/');
});
...
```
GET /auth/kakao로 카카오 로그인 과정 시작, 해당 라우터에서 로그인 전략을 수행, 그리고 로그인 성공 여부 결과를 /kakao/callback으로 받음 그리고 리다이렉트  
로컬 로그인과 다른 점은 passport.authenticate 메서드에 콜백 함수를 제공하지 않음, 내부적으로 req.login을 호출하기 때문에 직접 호출할 필요가 없음