# Sequelize 정리

## Sequlize란?
* 자바스크립트 구문을 알아서 SQL로 변환해주는 모듈
* 다른 데이터베이스 전환시에도 유용하게 사용가능

---

## Install Sequelize(Mysql)
```bash
npm install sequelize sequelize-cli mysql2
```
설치완료 후 ```npx seqeulize init``` 명령어 호출

## Sequelize 초기 생성 후 models/index.js 작성
```javascript
const Sequelize = require('sequelize');
const User = require('./user');
const Comment = require('./comment');

const env = process.env.NODE_ENV || 'development';
const config = require('../config/config');
const db = {};
// config는 시퀄라이즈 초기 생성 후 디비 연결파일


//새로운 시퀄라이즈 객체 생성 (객체 내부에는 디비 연결 정보를 가지고 있음)
const sequelize = new Sequelize(config.database, config.username, config.password,config);

db.sequelize = sequelize;

//db와 테이블 연결
db.User = User;
db.Comment = Comment;

User.init(sequelize);
Comment.init(sequelize);

User.associate(db);
Comment.associate(db);
//--------------------

module.exports = db;
```

## 시퀄라이즈를 이용하여 디비 연결 (app.js)
```javascript
...
const {sequelize} = require('./models');

//force : 서버 실행 시 마다 테이블을 재생성 할 것인지 아닌지
sequelize.sync({force: false})
    .then(()=>{
        cnosole.log("DB Connected Success");
    })
    .catch((err)=> {
        console.error(err);
    });
```

---

## 모델 생성
```javascript
const Sequelize = require('sequelize');

module.exports = class User extends Sequelize.Model {
    /*
        static init:
        테이블에 대한 자료형 지정 및 테이블 자체 설정

        static associate:
        테이블과 테이블의 관계에 대한 설정
    */
    static init(sequelize) {
        return super.init({
            //알아서 id 키 값을 생성하고 기본키로 만듬
            ...
            name: {
                type: Sequelize.STRING(20), //자료형 타입
                allowNull: false, //NULL 값 허용 여부
                unique: true, //UNIQUE 여부
            },
            age: {
                type: Sequelize.INTEGER.UNSIGNED,
                allowNull: false,
            },
            married: {
                type: Sequelize.BOOLEAN, // T/F
                allowNull: false,
            },
            ...
        }, {
            // 테이블에 대한 설정 지정
            sequelize,              // static init의 매개변수와 연결되는 옵션, model/index.js에서 연결
            timestamps: false,      // true시 createAt, updateAt 컬럼 추가 각각 생성 및 수정 시 시간 반영
            underscored: false,     // 테이블과 컬럼명을 자동으로 캐멀케이스로 만든다.
            modelName: 'User',      // 프로젝트에서 사용하는 모델의 이름
            tableName: 'users',     // 실제 데이터베이스의 테이블 이름
            paranoid: false,        // true로 설정 시 데이터 삭제 시 완벽하게 삭제하지 않고 삭제기록
            charset: 'utf8',
            collate: 'utf8_general_ci',
        });
    }
    static associate(db) {
        //테이블과 테이블의 관계를 설정
        
    }
}
```
---

## 관계 생성
모델의 ```static associate(db) {}``` 부분에서 테이블간의 관계를 작성

### __1:N__
```javascript
static associate(db) {
    db.User.hasMany(db.Comment, {foreignKey: 'commenter',sourceKey:'id'});
}
```
```javascript
static associate(db) {
    db.Comment.belongsTo(db.User, {foreignKey: 'commenter', targetKey:'id'});
}
```
다른 모델의 정보를 받아서 사용하는 테이블(외래키 컬럼이 존재하는)이 belongsTo를 사용함  
foreignKey : 저장할 외래 키  
targetKey : 외래키를 보유한 테이블 시점에서 원본 데이터를 보유한 테이블의 컬럼명  
sourceKey : 공유할 컬럼명

### __1:1__
```javascript
static associate(db) {
    db.User.hasOne(db.Comment, {foreignKey: 'UserId',sourceKey:'id'});
}
```
```javascript
static associate(db) {
    db.Comment.belongsTo(db.User, {foreignKey: 'UserId', targetKey:'id'});
}
```

### __N:M__

```javascript
static associate(db) {
    db.Post.belongsToMany(db.Hashtag,{through:'PostHashtag'});
}
```
```javascript
static associate(db) {
    db.Hashtag.belongsToMany(db.Post,{through:'PostHashtag'});
}
```

---
## Query
### CREATE
```javascript
//INSERT INTO nodejs.users (name, age,married,comment) VALUES ('zero',24,0,'자기소개1');
const {User} = require(../models);
User.create({
    name:'zero',
    age:24,
    married:false,
    comment: '자기소개1',
});
```

### READ
```javascript
//SELECT * FROM nodejs.users;
User.findAll({});

//SELECT * FROM nodejs.users LIMIT 1;
User.findOne({});

//SELECT name, married FROM nodejs.users;
User.findAll({
    attributes: ['name','married'],
});

/*
    Op 객체
    Op.gt:      초과
    Op.gte:     이상
    Op.lt:      미만
    Op.lte:     이하
    Op.ne:      같지않음
    Op.or:      또는
    Op.in:      배열 요소 중 하나
    Op.notIn:   배열 요소와 모두 다름
*/

//SELECT name, age FROM nodejs.users WHERE married=1 AND age > 30;
const {Op} = required('sequelize');
const {User} = required('../models');
User.findAll({
    attributes: ['name','age'],
    where: {
        married: true,
        age: {[Op.gt]: 30},
    },
});

//SELECT id, name FROM users WHERE married = 0 OR age > 30;
const {Op} = required('sequelize');
const {User} = required('../models');
User.findAll({
    attributes:['id','name'],
    where: {
        [Op.or]: [{married:false},{age:{[Op.get]:30}}],
    },
});

//SELECT id, name FROM users ORDER BY age DESC;
User.findAll({
    attributes: ['id','name'],
    order: [['age','DESC']],
});

//SELECT id, name FROM users ORDER BY age DESC LIMIT 1;
User.findAll({
    attributes: ['id','name'],
    order: [['age','DESC']],
    limit: 1,
});

//SELECT id, name FROM users ORDER BY age DESC LIMIT 1 OFFSET 1;
User.findAll({
    attributes:['id','name'],
    order: ['age','DESC'],
    limit: 1,
    offset: 1,
});
```

### UPDATE
```javascript
//UPDATE nodejs.users SET comment = 'Change' WHERE id = 2;
User.update({
    comment: 'Change',
    where: {
        id: 2,
    },
});
```

### DELETE
```javascript
User.destory({
    where: {id:2},
});
```
---
## 관계 쿼리
include 속성을 사용하여 외래키로 연결되어있는 테이블의 데이터까지 반환한다.
```javascript
const user = await User.findOne({
    include: [{
        model:Comment,
    }]
});
console.log(user.Comments);

//같은 방법
const user = await User.findOne({});
const comments = await user.getComments();
console.log(comments);

/*
    관계가 설정되어있다면, getComments, setComments, addComment, addComments, removeComments 메서드를 지원한다.
    include 내에도 where나 attribute를 지원한다.
*/
```
