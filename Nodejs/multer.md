# multer
이미지 동영상 등을 비롯한 여러 가지 파일들을 멀티파트 형식으로 업로드 할 때 사용하는 미들웨어

## 설치
```npm i multer```

## 기본 설정
```javascript
const multer = require('multer');

//multer를 사용하기 위해서는 uploads 디렉터리가 필요하다.
try {
    fs.readdirSync('uploads');
} catch (error) {
    console.error('uploads 폴더가 없어 uploads 폴더를 생성합니다.');
    fs.mkdirSync('uploads');
}

const upload = multer({
    stroage: multer.diskStorage({
        destination(req,file,done) {
            done(null, 'uploads/');
        },
        filename(req,file,done) {
            const ext = path.extname(file.originalname);
            done(null,path.basename(file.originalname,ext) + Date.now() + ext);
        },
    }),
    limits: {fileSize: 5 * 1024 * 1024},
});

//파일 업로드 라우터
/*
    upload.single => 한개의 파일만 업로드
    upload.array('many') => 여러개의 파일을 업로드
    upload.none() => 파일 업로드 없음   
*/
app.post('/uploads',upload.single('image'),(req,res)=> {
    console.log(req.file, req.body);
    res.send('ok');
});
```
multer 함수의 인수로 설정을 넣음,  
storage:  
어디에 (destination) 어떤 이름으로(filename)으로 저장할지 결정  
파라미터의 req는 요청에 대한 정보, file은 업로드 파일에 대한 정보, done은 함수  
limits에는 파일 크기에 대한 제한  
  
라우터 미들웨어 앞에 넣어두면 multer 설정에 따라 파일 업로드 후 req.file 객체가 생성. 인수는 input 태그의 name이나 form 데이터의 키와 일치