# 💡여행 플랜 서비스 플랫폼(SunTour)



## 1. 제작 기간 & 참여 인원
- 2023년 4월 ~ 5월
- 5명



## 2. 사용 기술
#### `Back-end`
- Spring-framework(MVC) 5.2.18 RELEASE
- Mybatis 3.5.4
- Java 1.8.0

#### `Front-end`
- HTML5 
- CSS3 
- javaScript ES6
- J-Query 3.6.0
- JSP 
#### `Server`
- Apache-Tomcat 9.0.71 
#### `Database`
- My-SQL 8.0.28   


## 3. 내 역할과 업무성과
- 공통개발 역할(백엔드)
- AWS S3 - AWS IAM, S3 bucket 생성 및 업로드 공통 클래스 작성
- 아임포트 - 결제 API  
- 인터셉터 - Auth 체크용 어노테이션 구현
- 웹소켓 STOMP - 플래너와 유저간 채팅 구현
- 게시판 CRUD - 플랜 게시판 구현
- 카트 CRUD - 카트 서비스 구현

## 4. 구현 기능 코드 
<details>
<summary><b>구현 기능 설명 펼치기</b></summary>
<div markdown="1">

### 4.1. 전체 흐름

![image](https://user-images.githubusercontent.com/120711406/235872521-33d3533d-7baf-4a72-9449-1253a5e2006d.png)

	
---
	
	
	
### 4.2. AWS S3 - AWS IAM, S3 bucket 생성 및 업로드 공통 클래스 작성

![image](https://user-images.githubusercontent.com/120711406/235873702-5127c63d-19e5-406b-8919-01dd323d2255.png)

<details>
<summary> <b>IAM 권한설정</b> </summary>
	
- IAM 사용자 생성
- 권한으로 AmazonS3FullAccess 추가
	
![image](https://user-images.githubusercontent.com/120711406/235908642-a1dbf375-e3ad-4c73-a6bb-b291ad0f3e58.png)
	
</details>
	
<details>
<summary> <b>버킷 정책 생성</b> </summary>
- 버킷을 사용하기 위해 정책생성
	
![image](https://user-images.githubusercontent.com/120711406/235909022-146e7ec1-4f9d-4f64-a8ec-326a74e954a5.png)
	
</details>
	
<details>
<summary> <b>공통 클래스 구현</b> </summary>
- 이미지 다중 업로드, 삭제 를 위한 공통 클래스를 구현.
	
```java
@Service
public class S3FileUploadService {

    @Autowired
    private final AmazonS3Client amazonS3Client; //아마존 계정정보 propertie파일 -> common-context에서 주입
    @Value("${aws.s3.bucket}")
    private String bucket; //S3버킷정보
    @Value("${aws.s3.bucket.url}") //지역정보
    private String defaultUrl;

    public S3FileUploadService(AmazonS3Client amazonS3Client) {
        this.amazonS3Client = amazonS3Client;
    }

    //생성자 주입
    public List<String> upload(List<MultipartFile> uploadFile) throws IOException {
        List<String> urlList = new ArrayList<>(); //업로드된 url을 받기위한 리스트

        //파일이름 새로만들어서 리스트에 담기
        List<Map<String, String>> fileList = new ArrayList<>();
        for (int i = 0; i < uploadFile.size(); i++) {
            String origName = uploadFile.get(i).getOriginalFilename(); //원 파일이름
            String ext = origName.substring(origName.lastIndexOf('.')); // 확장자
            String saveFileName = getUuid() + ext; //uuid로 새이름 만들기
            Map<String, String> map = new HashMap<>();
            map.put("saveFile", saveFileName);
            fileList.add(map);
        }

        for (int i = 0; i < uploadFile.size(); i++) {
            String url = "";
            File file = new File(System.getProperty("user.dir") + fileList.get(i).get("saveFile"));
            //로컬 현재위치에 임시저장 객체 만듬
            uploadFile.get(i).transferTo(file); //로컬에 파일 임시저장
            uploadOnS3(fileList.get(i).get("saveFile"), file); //업로드
            url = defaultUrl + '/' + fileList.get(i).get("saveFile"); //업로드한 파일의 url주소
            urlList.add(url); //리턴을 위해 담음
            file.delete(); // 임시파일 삭제
        }
        return urlList; //업로드 후 리턴값 (List<String> 타입)
    }

    // UUID만드는 메소드(중간의-는 지워줌)
    private static String getUuid() {
        return UUID.randomUUID().toString().replaceAll("-", "");
    }

    //S3업로드 메소드
    private void uploadOnS3(final String findName, final File file) {
        // AWS S3 전송 객체 생성
        final TransferManager transferManager = new TransferManager(this.amazonS3Client);
        // 요청 객체 생성
        final PutObjectRequest request = new PutObjectRequest(bucket, findName, file);
        // 업로드 시도
        final Upload upload = transferManager.upload(request);

        try {
            upload.waitForCompletion();
        } catch (AmazonClientException | InterruptedException amazonClientException) {
            amazonClientException.printStackTrace();
        }
    }
    //S3 객체 삭제 메소드
    public void deleteFromS3(final String findName) {
        String realFileName = findName.substring(53);
        // 삭제할 객체 생성
        final DeleteObjectRequest deleteRequest = new DeleteObjectRequest(bucket, realFileName);
        // 삭제
        this.amazonS3Client.deleteObject(deleteRequest);
    }

}

```
	
</details>

<details>
<summary> <b>Controller</b> </summary>

- Plan 게시판 Controller 업로드

```java 
  @PostMapping("create")
    public String planPut(PlanDTO planDTO, ImgDTO imgDTO, HttpSession httpSession,
                          @RequestParam("files[]") List<MultipartFile> multipartFile) throws IOException {
        String user = (String) httpSession.getAttribute("user_id");
        planDTO.setUser_id(user);
        int plan_idx = planService.planCreate(planDTO); // 게시글 생성
        if(plan_idx!=0){ // 이미지 파일 생성
            if(multipartFile !=null || !multipartFile.isEmpty()){ // 이미지 파일 있으면
                List<String> imgUrlList = s3FileUploadService.upload(multipartFile); // 서버에 이미지 파일 저장 후 URL값 List에 담기
                planDTO.setPlan_idx(plan_idx); // 게시글 인덱스 set
                planDTO.setP_img(imgUrlList); // 이미지 url set
                boolean success = this.planService.planImgCreate(planDTO); // 이미지 저장 성공
                if(success){
                    return "redirect:/plan/list";
                }
            }
        }
        return "/plan/plan_create";
    }
```
</details>

<details>
<summary> <b>설정</b> </summary>

- 라이브러리 설치
- Key 노출을 피하기 위해 properties 파일 등록 후 클래스 빈설정 생성자 값으로 설정
 
```xml
	   <constructor-arg>
            <bean class="com.amazonaws.auth.BasicAWSCredentials">
                <constructor-arg value="${aws.accessKey}"/>
                <constructor-arg value="${aws.secretKey}"/>
            </bean>
        </constructor-arg>
    </bean>
    <bean id="awsProperties" class="org.springframework.beans.factory.config.PropertiesFactoryBean">
        <property name="location" value="classpath:common.properties"/>
    </bean>
    <bean class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
        <property name="properties" ref="awsProperties"/>
    </bean>
```
</details>

<details>
<summary> <b>어려웠던 점</b> </summary>
	
- AWS를 처음 다루게되어 개념이해에 어려움이 있었음.
- AWS는 업데이트가 빠르기 때문에 최신 정보를 찾기가 힘들었음. (대부분의 메뉴가 변경되었음)
- Spring Legacy 프로젝트는 Spring boot 에 비해 properties나 yalm파일을 활용하기 복잡했음.
- IAM 사용자 키가 깃허브에 노출되었을땐 AWS에서 메일로 경고만 주는것 뿐만 아니라, 권한을 변경해버린다.
- AWS에서는 키가 노출되었을 경우, 사용자 삭제후 재생성을 추천한다. (키발급만 다시하는것 보다) (항상 주의하자)

</details>
  
<details>
<summary> <b>앞으로 해야될 것</b> </summary>
	
- AWS RDS 테스트중 추가 결제가 되었음. 학습이 더 필요함.
- 깃허브 액션과 S3 EC2 연계로 CICD구현(진행중)
- EC2 학습 진행중 리눅스 학습의 필요성을 느낌.
	
</details>


---	
	
	
	
### 4.3. 아임포트 결제 API 

![image](https://user-images.githubusercontent.com/120711406/235916623-f8144c4f-73a0-4765-86eb-a0d9c3f4c2b4.png)
	
<details>
<summary> <b>공통 클래스 구현</b> </summary>

- 실제 결제한 가격이 고지된 가격과 동일한지 검증
- 검증후 결제정보를 DB에 저장
	
```java
@RestController
public class PaymentController {
    private final IamportClient iamportClient;
    private final PaymentService paymentService;

    public PaymentController(IamportClient iamportClient, PaymentService paymentService) {
        this.iamportClient = iamportClient;
        this.paymentService = paymentService;
    }

    // 결제 서버검증(실제 결제한 가격이 고지된 가격과 동일한지 검증)
    @PostMapping("/verifyIamport/{imp_uid}")
    public IamportResponse<Payment> paymentByUid(@PathVariable(value = "imp_uid") String imp_uid) throws IamportResponseException, IOException {
        return iamportClient.paymentByImpUid(imp_uid);
    }

    // 결제정보 DB입력
    @PostMapping(value = "/payment/confirm", consumes = "application/json")
    public Map<String, Object> paymentConfirm(@RequestBody PayDTO payDTO) {
        System.out.println(payDTO.toString());
        boolean checkPayment = paymentService.pay(payDTO);
        Map<String, Object> map = new HashMap<String, Object>();
        if (checkPayment) {
            paymentService.saleCount(payDTO);
            map.put("msg", "결제성공");
        } else {
            map.put("msg", "결제실패");
        }
        return map;
    }
}
```
	
</details>
  
<details>
<summary> <b>JavaScript</b> </summary>

  
- 아임포트 결제, ajax 콜백함수 구현

```javascript
// 2023.04.23 길영준
// 카카오페이 결제
    const price = $('#price').val(); // 가격
    const name = $('#title').val(); //플랜명
    const buyer = $('.session').val(); //구매자아이디
    const planner = $('#planner').val(); // 플래너아이디
    const plan_idx = $('.plan_idx').val(); //플랜 pk
    // 아임포트 결제 함수
    function kakao() {
        let IMP = window.IMP;
        IMP.init('imp67107132');
        IMP.request_pay({
            pg: 'kakaopay.TC0ONETIME',
            merchant_uid: 'suntour_' + new Date().getTime(), //상점에서 생성한 고유 주문번호
            name: name, // 상품명
            amount: price, // 가격
            buyer_name: buyer // 구매자
        }, function (rsp) { // 검증 로직
            $.ajax({
                type: 'POST',
                url: '/verifyIamport/' + rsp.imp_uid
            }).done(function (result) {
                if (rsp.paid_amount === result.response.amount) {
                    let info = {
                        imp_uid: rsp.imp_uid,
                        merchant_uid: rsp.merchant_uid,
                        buyer_id: buyer,
                        planner_id: planner,
                        plan_idx: plan_idx
                    }
                    $.ajax({//결제 검증 ajax
                        type: 'POST',
                        data: JSON.stringify(info),
                        url: '/payment/confirm',
                        dataType: "json",
                        contentType: 'application/json; charset=utf-8',
                        success: function (result) {
                            alert(result.msg)
                            window.location.reload();
                        },
                        error: function (xhr, status, error) {
                            alert(result.msg)
                            console.log(xhr)
                            console.log(status)
                            console.log(error)
                        }
                    })
                } else {
                    alert("결제실패" + "에러 : " + rsp.error_code + "에러내용: " + rsp.error_msg);
                }
            })

        });
    }

```
</details>

<details>
<summary> <b>설정</b> </summary>

- 라이브러리 설치
- CDN 적용
- Key 노출을 피하기 위해 properties 파일 등록 후 클래스 빈설정 생성자 값으로 설정
	
```xml
    <bean id="iamport" class="com.siot.IamportRestClient.IamportClient">
        <constructor-arg index="0" value="${iamport.api}"/>
        <constructor-arg index="1" value="${iamport.api_secret}"/>
    </bean>
```
</details>

<details>
<summary> <b>어려웠던 점</b> </summary>

- 아임포트 CDN 버전업 업데이트 내역을 뒤늦게 확인. (더이상 지원하지 않는 파라미터)
- 초반에 성급하게 진행하여, 구조를 잘못 이해함.
- 아임포트에서 발행하는 secret id와 key는 클라이언트 결제정보를 결제사에서 가져오기 위해 있음.
	
</details>
	
<details>
<summary> <b>앞으로 해야될 것</b> </summary>
   
- 더 다양한 API를 사용해 볼것
- 이를 통해 메뉴얼을 이해하고 응용해볼것.
- JavaScript만으로는 왜 데이터조작에 더 취약한지 학습해 볼것.
- 구현 전 API의 버전과 그에맞는 내용을 먼저 인지할 것.

</details>
	

---	
	
	
	
### 4.4. 인터셉트를 활용한 권한 체크 용도 어노테이션 구현

```java
 @Auth(role = Auth.Role.ADMIN)
```

<details>
<summary> <b>기능 설명</b> </summary>
	
- 세션으로 권한체크를 매번 해주는 불편함을 덜기 위해 작성
- 이 어노테이션으로 권한별 메소드 실행(라우팅)이 가능.

 
</details>
  
<details>
<summary> <b>어노테이션 클래스</b> </summary>
	
- Retention : 라이프사이클을 런타임중에만 으로 설정
- Target : 메소드에 어노테이션을 적용시킴
	
```java
@Retention(RUNTIME)
@Target(METHOD)
public @interface Auth {
    public enum Role {ADMIN, USER, PLANNER}

    public Role role() default Role.USER;
}

```
	
</details>
	  
<details>
<summary> <b>인터셉터</b> </summary>

- preHandle 메소드를 오버라이딩 하여 컨트롤러로 가기전에 권한체크를 할 수 있다.
- getMethodAnnotaion 메소드로 만들어둔 권한 어노테이션 클래스를 지정한다.
- 세션에서 받아오는 권한값을 기준으로 조건식을 주어 True는 실행 False는 redirect를 시킨다.

```java
public class AuthInterceptor extends HandlerInterceptorAdapter {
    @Override
    public boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws Exception {
        if (!(handler instanceof HandlerMethod)) {
            return true; //메소드핸들러가 아닐때 실행시킴
        }
        HandlerMethod handlerMethod = (HandlerMethod) handler;

        Auth auth = handlerMethod.getMethodAnnotation(Auth.class); //어노테이션클래스 지정
        if (auth == null) {
            return true;    //어노테이션 지정되지 않았으면 실행시킴
        }

        HttpSession httpSession = request.getSession();
        if (httpSession == null) {
            response.sendRedirect(request.getContextPath() + "/user/signin");
            return false; //어노테이션은 있으나 세션이 없으면 리다이렉트
        }
        String authUser = (String) httpSession.getAttribute("auth");
        if (authUser == null) {
            response.sendRedirect(request.getContextPath() + "/user/signin");
            return false; // 세션에 auth 값이 없으면 리다이렉트
        }
        String role = auth.role().toString();
        if ("ADMIN".equals(role)) {
            if (!"auth_a".equals(authUser)) {
                response.sendRedirect(request.getContextPath() + "/user/signin");
                return false; // 롤이 ADMIN 이 아니면 리다이렉트
            }
        }
        if ("PLANNER".equals(role)) {
            if ("auth_a".equals(authUser)) {
                return true; //롤이 어드민이면 통과
            }
            if (!"auth_b".equals(authUser)) {
                response.sendRedirect(request.getContextPath() + "/user/signin");
                return false; //롤이 플래너가 아니면 통과시키지 않음
            }
        }
        return true;    //해당조건이 false가 아니면 진행시킴
    }
}

```
	
</details>
	
<details>
<summary> <b>설정</b> </summary>

- Servlet-context 에 해당 인터셉터를 등록해준다.
	  
```xml
    <interceptors>
        <interceptor>
            <mapping path="/**"/>
            <beans:bean id="authInterceptor" class="com.goott.pj3.common.util.auth.AuthInterceptor"/>
        </interceptor>
    </interceptors>
```

</details>


<details>
<summary> <b>어려웠던 점</b> </summary>

- 간단한 조건 같았지만 생각보다 쓰임을 더 고려해야 했다.
- Target이 메소드가 아닌 클래스로 작성하려 해보았으나, admin Controller의 경우에도 때에따라 필요로 하는 권한이 달랐다.

</details>
	
<details>
<summary> <b>앞으로 해야될 것</b> </summary>

- 쓰임이 반복되는 기능은 어노테이션 작성 으로 대체 가능한지 고려해볼 것.
- preHandle 이외에 postHandle, afterCompletion 도 활용가능한 기능이 있는지 고려해 볼 것.
	
</details>

	
---


	
### 4.5. 웹소켓 STOMP를 활용한 채팅 구현
	
- UI, UX 구현중(2023.05.03 기준)

![image](https://user-images.githubusercontent.com/120711406/235933242-b170f3ec-0c6a-49a2-aa55-1919415c4853.png)

![image](https://user-images.githubusercontent.com/120711406/235933551-b09481a4-da94-4e4f-99b8-c0da315215d3.png)
	
![image](https://user-images.githubusercontent.com/120711406/235934373-a55dfd79-0c46-4100-b829-ae9cfa780935.png)


<details>
<summary> <b>기능 설명</b> </summary>

  - 유저와 판매자 간의 1:1 채팅방 구현
  - 채팅방 목록, 대화로그 저장, 대화 조회 여부 확인, 새로운 메세지 도착 알림 구현

</details>
  
<details>
<summary> <b>STOMP 웹소켓 설정 클래스</b> </summary>
  
- 엔드포인트와 publish, subscribe 값 설정
- 소켓JS 사용 설정

```java
@Configuration
@EnableWebSocketMessageBroker//Stomp를 사용하기 위해 선언
public class StompWebSocketConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/stomp/chat") //엔드포인트
                .setAllowedOrigins("http://localhost:8080")
                .withSockJS();
    }

    /*어플리케이션 내부에서 사용할 path를 지정할 수 있음*/
    @Override
    public void configureMessageBroker(MessageBrokerRegistry registry) {
        registry.setApplicationDestinationPrefixes("/pub"); //클라이언트에서 SEND요청을 처리
        registry.enableSimpleBroker("/sub"); //경로에 SimpleBroker를 등록 
                                                            // 해당 경로를 Subscribe하는 client에게 메시지를 전달
        //.enableStompBrokerRelay = SimpleBroker의 기능과 외부 Message Broker( RabbitMQ, ActiveMQ 등 )에 메세지를 전달
    }
}
```
	
</details>

<details>
<summary> <b>Chat Controller</b> </summary>

- 메세지 매핑으로 해당 구독url로 메세지를 전달해준다.
- DTO를 DB에 전달, 메세지 로그를 저장한다.
- 메세지 도착 실시간 알림을 구독 url로 전달해 준다.

```java
@Controller
public class StompChatController {
    private final SimpMessagingTemplate template; //특정 Broker로 메세지를 전달
    private final ChatRoomRepository repository;


    public StompChatController(SimpMessagingTemplate template, ChatRoomRepository repository) {
        this.template = template;
        this.repository = repository;
    }

    //Client가 SEND할 수 있는 경로
    //stompConfig에서 설정한 applicationDestinationPrefixes와 @MessageMapping 경로가 병합됨
    //"/pub/chat/enter"
    @MessageMapping(value = "/chat/enter")
    public void enter(ChatMessageDTO chatMessageDTO) {
        chatMessageDTO.setMsg_content(chatMessageDTO.getSend_id() + "님이 채팅방에 참여하였습니다.");
        template.convertAndSend("/sub/chat/room/" + chatMessageDTO.getMsg_idx(), chatMessageDTO);
    }

    @MessageMapping(value = "/chat/message") //DTO = roomid, message, 보낸사람, 받는사람
    public void message(ChatMessageDTO chatMessageDTO) {
        template.convertAndSend("/sub/chat/room/" + chatMessageDTO.getMsg_idx(), chatMessageDTO);
        repository.saveMessageLog(chatMessageDTO);  //로그 DB에 저장
        //실시간 알람
        String alarmDestination = "/sub/chat/alarm/" + chatMessageDTO.getReceive_id();
        String alarmMessage = chatMessageDTO.getSend_id() + "님의 새로운 메세지";
        template.convertAndSend(alarmDestination, alarmMessage);
    }
}
```

</details>
  
<details>
<summary> <b>Room Controller</b> </summary>

- 채팅방 개설, 실제 채팅방, 목록조회 구현
- 여러 조건을 사용해 어뷰징을 차단

```java
@RequestMapping(value = "/chat")
@Controller
public class RoomController {

    private final ChatRoomRepository repository;

    public RoomController(ChatRoomRepository repository) {
        this.repository = repository;
    }

    //채팅방 목록 조회
    @GetMapping(value = "/rooms/{user_id}")
    public ModelAndView rooms(@PathVariable("user_id") String user_id, HttpSession httpSession, ModelAndView mv) {
        String sessionId = String.valueOf(httpSession.getAttribute("user_id"));
        if (sessionId.equals(user_id)) { //뷰에서 넘어온 user_id와 session user_id를 비교해서 일치하면 채팅방 목록을 보여줌
            mv.setViewName("/plan/rooms");
            if (repository.checkReadOrNot(sessionId) != null) {
                mv.addObject("YorN", repository.checkReadOrNot(sessionId)); // 읽지않은 메세지가 있는지 DB에서 확인
            }
            mv.addObject("list", repository.findAllRooms(sessionId)); //세션아이디가 가지고 있는 모든 채팅방 리스트 가져오기
        } else {
            mv.setViewName("redirect:/user/signin");    //일치하지 않으면 로그인페이지로 보냄
        }
        return mv;
    }

    //채팅방 개설
    @PostMapping(value = "/room") //form으로 받는데이터 = send_id & receive_id
    public String create(ChatRoomDTO chatRoomDTO, ModelAndView mv) {
        if (chatRoomDTO.getSend_id().equals(chatRoomDTO.getReceive_id())) {
            return "redirect:/plan/list";   // 플래너가 본인에게 채팅 요청했을시
        }
        ChatRoomDTO formData = chatRoomDTO; // 폼에서 받아온 dto
        System.out.println("폼으로 받아온 dto : " + formData.toString());

        if (repository.findRoomByName(formData) != null) {  //이미 해당 플래너와 채팅방이 존재하면 존재하는 방으로 이동시킴
            int msg_idx = repository.findRoomByName(formData).getMsg_idx();
            System.out.println("방이 존재할때 가져온 방 idx : " + msg_idx);
            return "redirect:/chat/room/" + msg_idx;
        } else {                                             //없다면 새로 생성해주고 방으로 이동
            repository.createChatRoomDTO(formData);
            int msg_idx = chatRoomDTO.getMsg_idx();
            System.out.println("방만들고 받아온 idx : " + chatRoomDTO.getMsg_idx());
            return "redirect:/chat/room/" + msg_idx;
        }
    }

    // 실제 채팅방
    @GetMapping("/room/{msg_idx}")
    public String getRoom(@PathVariable("msg_idx") int msg_idx, Model model, HttpSession httpSession) {
        String sessionAuth = String.valueOf(httpSession.getAttribute("auth"));
        String user = "";
        String planner = "";
        if (sessionAuth.equals("auth_c")) { //무분별한 겟요청으로 채팅방 열람을 막기위해 세션아이디를 가져옴
            user = String.valueOf(httpSession.getAttribute("user_id")); // 유저 일때 아이디
            System.out.println("유저아이디 : " + user);
        } else if (sessionAuth.equals("auth_b")) {
            planner = String.valueOf(httpSession.getAttribute("user_id"));  // 플래너 일때 아이디
            System.out.println("플래너아이디 : " + planner);
        } else {
            return "redirect:/main"; // 둘다 아니면 메인으로
        }
        ChatRoomDTO chatRoomDTO = new ChatRoomDTO();
        chatRoomDTO.setMsg_idx(msg_idx);
        chatRoomDTO = repository.findRoomById(chatRoomDTO);
        if (chatRoomDTO.getSend_id().equals(user)
                || chatRoomDTO.getReceive_id().equals(planner)) { // 보낸아이디와 세션유저아이디가 맞거나
            int roomID = chatRoomDTO.getMsg_idx();
            if (repository.findMessageLog(roomID) != null) {  //로그를 찾아왔을때
                //세션값이 receive_id 일때 N-> Y 메세지 읽음 표시
                Map<String, String> map = new HashMap<>();
                map.put("msg_idx", String.valueOf(roomID));
                map.put("session_id", String.valueOf(httpSession.getAttribute("user_id")));
                repository.readNtoY(map);
                model.addAttribute("chatLog", repository.findMessageLog(roomID)); //로그 불러오기
                model.addAttribute("room", chatRoomDTO);  // 받은아이디와 세션플래너아이디가 맞으면
                return "/plan/room";
            } else { //로그가 없을때
                model.addAttribute("room", chatRoomDTO);  // 받은아이디와 세션플래너아이디가 맞으면
                return "/plan/room";
            }
        } else {
            return "redirect:/main";                            // 아닐경우 메인으로
        }
    }
} 
```

</details>
  
<details>
<summary> <b>Repository</b> </summary>

- 모든 채팅방조회, 로그저장, 로그조회, 조건과 일치하는 채팅방 조회 등을 구현한다.

```java
@Repository
public class ChatRoomRepository {
    private Map<String, ChatRoomDTO> chatRoomDTOMap;
    final
    SqlSession session;

    public ChatRoomRepository(SqlSession session) {
        this.session = session;
    }

    //채팅방 만들기
    public void createChatRoomDTO(ChatRoomDTO chatRoomDTO) {
        session.insert("chat.create", chatRoomDTO);
    }

    // 소유하고있는 모든 채팅방 리스트 가져오기
    public List<ChatRoomDTO> findAllRooms(String user_id) {
        return session.selectList("chat.findAllRooms", user_id);
    }

    // 채팅방ID로 채팅찾기
    public ChatRoomDTO findRoomById(ChatRoomDTO chatRoomDTO) {
        return session.selectOne("chat.findRoomById", chatRoomDTO);
    }

    // 보내는 사람 받는사람 이름으로 채팅방이 이미 존재하는지 확읺하고
    // 있다면 채팅방ID를 리턴한다
    public ChatRoomDTO findRoomByName(ChatRoomDTO chatRoomDTO) {
        return session.selectOne("chat.findRoomByName", chatRoomDTO);
    }

    //메세지로그 저장
    public void saveMessageLog(ChatMessageDTO chatMessageDTO) {
        session.insert("chat.saveMessageLog", chatMessageDTO);
    }
    //메세지로그 불러오기
    public List<ChatMessageDTO> findMessageLog(int msg_idx) {
        return session.selectList("chat.findMessageLog", msg_idx);
    }
    //읽었나 안읽었나 확인
    public void readNtoY(Map<String, String> map) {
        session.update("chat.readNtoY", map);
    }
    // 채팅방 생성시 안읽은 메세지가 있는방 표시
    public List<ChatRoomDTO> checkReadOrNot(String sessionId) {
        ChatRoomDTO chatRoomDTO = new ChatRoomDTO();
        System.out.println(session.selectList("chat.checkReadorNot", sessionId));
        chatRoomDTO.setReceive_id(sessionId);
        return session.selectList("chat.checkReadorNot", chatRoomDTO);
    }
}
```

</details>



<details>
<summary> <b>SQL</b> </summary>

- mybatis 활용

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="chat">
    <insert id="create" parameterType="com.goott.pj3.common.util.chat.ChatRoomDTO"
            useGeneratedKeys="true" keyProperty="msg_idx">
        INSERT INTO msg (send_id, receive_id, msg_img, msg_content)
        SELECT #{send_id}, #{receive_id}, '', ''
        FROM DUAL
        WHERE NOT #{send_id} = #{receive_id}
          AND NOT EXISTS (SELECT msg_idx
                          FROM msg
                          WHERE (send_id = #{send_id} AND receive_id = #{receive_id})
                             OR (send_id = #{receive_id} AND receive_id = #{send_id}))
    </insert>
    <!--방id로 채팅방 찾기-->
    <select id="findRoomById" resultType="com.goott.pj3.common.util.chat.ChatRoomDTO">
        SELECT msg_idx, send_id, receive_id, create_date
        FROM msg
        WHERE msg_idx = #{msg_idx}
    </select>
    <!--해당 유저의 모든 방 찾기-->
    <select id="findAllRooms" resultType="com.goott.pj3.common.util.chat.ChatRoomDTO">
        SELECT msg_idx, send_id, receive_id, msg_img, msg_content, create_date
        FROM msg
        WHERE send_id = #{user_id}
           OR receive_id = #{user_id}
        ORDER BY msg_idx DESC
    </select>
    <!--유저이름으로 채팅방 찾기-->
    <select id="findRoomByName" resultType="com.goott.pj3.common.util.chat.ChatRoomDTO">
        SELECT msg_idx, send_id, receive_id
        FROM msg
        WHERE (send_id = #{send_id} AND receive_id = #{receive_id})
           OR (send_id = #{receive_id} AND receive_id = #{send_id})
    </select>
    <!--메세지 로그 저장 (이미지는 추후)-->
    <insert id="saveMessageLog" parameterType="com.goott.pj3.common.util.chat.ChatMessageDTO">
        INSERT INTO msg_log(msg_idx, send_id, receive_id, msg_content, msg_img)
        VALUES (#{msg_idx}, #{send_id}, #{receive_id}, #{msg_content}, '없음')
    </insert>
    <!--메세지 로그 찾아오기-->
    <select id="findMessageLog" resultType="com.goott.pj3.common.util.chat.ChatMessageDTO">
        SELECT msg_idx, send_id, receive_id, msg_img, msg_content, create_date, read_yn
        FROM (SELECT msg_idx, send_id, receive_id, msg_img, msg_content, create_date, read_yn
              FROM msg_log
              WHERE msg_idx=#{msg_idx}
              ORDER BY create_date DESC
              LIMIT 10) as sub
        ORDER BY create_date ASC
    </select>
    <!--읽었나 확인-->
    <update id="readNtoY">
        UPDATE  msg_log
        SET read_yn = 'y'
        WHERE msg_idx = #{msg_idx} AND receive_id = #{session_id}
    </update>
    <!--채팅방 리스트 생성시 안읽은 메세지가 있는 방을 표시해줌-->
    <select id="checkReadorNot" resultMap="msgidxResultMap">
        SELECT msg_idx
        FROM msg_log
        WHERE receive_id = #{receive_id}
          AND read_yn = 'n'
    </select>
    <resultMap id="msgidxResultMap" type="com.goott.pj3.common.util.chat.ChatRoomDTO">
        <collection property="msg_idx" column="msg_idx" javaType="List" ofType="Integer">
            <result column="msg_idx"/>
        </collection>
    </resultMap>
</mapper>

```

</details>
	  
<details>
<summary> <b>DTO</b> </summary>

- 메세지를 위한 DTO, 채팅방을 위한 DTO 2개를 작성.

</details>
	
<details>
<summary> <b>JSP</b> </summary>
	
- 방 목록
	
```jsp
<c:forEach items="${list}" var="room">
                <c:if test="${sessionScope.user_id == room.send_id}">
                    <li><a href="/chat/room/${room.msg_idx}" id="room-name">${room.receive_id} 와 대화하기</a></li>
                    <div id="msgArea"></div>
                    <p>채팅 생성날짜 :${room.create_date}</p>
                    <c:forEach items="${YorN}" var="test">
                        <c:if test="${test.msg_idx eq room.msg_idx}">
                            <p>읽지않은 메세지가 있습니다.</p>
                        </c:if>
                    </c:forEach>
                </c:if>
                <c:if test="${sessionScope.user_id == room.receive_id}">
                    <li><a href="/chat/room/${room.msg_idx}" id="room-name2">${room.send_id} 와 대화하기</a></li>
                    <div id="msgArea"></div>
                    <p>채팅 생성날짜 :${room.create_date}</p>
                    <c:forEach items="${YorN}" var="test">
                        <c:if test="${test.msg_idx eq room.msg_idx}">
                            <p>읽지않은 메세지가 있습니다.</p>
                        </c:if>
                    </c:forEach>
                </c:if>
            </c:forEach>
```
	
- 채팅방

```jsp
        <c:forEach var="log" items="${chatLog}">
            <c:if test="${log.send_id == sessionScope.user_id}">
                <div class='col-6'>
                    <div class='alert alert-secondary'>
                        <b> ${log.send_id} : ${log.msg_content}</b>
                        <fmt:formatDate pattern="MM-dd HH:mm" value="${log.create_date}"/>
                        <p>${log.read_yn}</p>
                    </div>
                </div>
            </c:if>
            <c:if test="${log.send_id != sessionScope.user_id}">
                <div class='col-6'>
                    <div class='alert alert-warning'>
                        <b> ${log.send_id} : ${log.msg_content} </b>
                        <fmt:formatDate pattern="MM-dd HH:mm" value="${log.create_date}"/>
                    </div>
                </div>
            </c:if>
        </c:forEach>
```

</details>
	
<details>
<summary> <b>JavaScript</b> </summary>
	
- 방 목록
	
```javascript
    let alarmLaunched = false;
    $(document).ready(function () {
        let sockJs = new SockJS("/stomp/chat");
        let stomp = Stomp.over(sockJs);
        stomp.connect({}, function () {
            console.log("STOMP Connection")
            stomp.subscribe("/sub/chat/alarm/" + '${sessionScope.user_id}', function (chat) {
                if (!alarmLaunched) {
                    let msg = chat.body
                    console.log(msg)
                    var str = `<div class='col-6'><div class='alert alert-secondary'><input id="alert"  value="\${chat.body}\"></div></div>`;
                    $("#msgArea").append(str);
                    alarmLaunched = true;
                }
            });
        });
    });
```

- 채팅방

```javascript
    $(document).ready(function () {

        let roomId = '${room.msg_idx}';
        let username = '${sessionScope.user_id}';
        let receiveName = '';
        if (username === '${room.send_id}') {
            receiveName = '${room.receive_id}';
        } else {
            receiveName = '${room.send_id}';
        }

        console.log(roomId + ", " + username);

        let sockJs = new SockJS("/stomp/chat");
        //1. SockJS를 내부에 들고있는 stomp를 내어줌
        let stomp = Stomp.over(sockJs);

        //2. connection이 맺어지면 실행
        stomp.connect({}, function () {
            console.log("STOMP Connection")

            //4. subscribe(path, callback)으로 메세지를 받을 수 있음
            stomp.subscribe("/sub/chat/room/" + roomId, function (chat) {
                var content = JSON.parse(chat.body);

                var writer = content.send_id;
                var str = '';

                    let date = new Date().toLocaleString()
                if (writer === username) {
                    str = "<div class='col-6'>";
                    str += "<div class='alert alert-secondary'>";
                    str += "<b>" + writer + " : " + content.msg_content +"</b>";
                    str += "<p>" + date + "</p>"
                    str += "</div></div>";
                    $("#msgArea").append(str);
                } else {
                    str = "<div class='col-6'>";
                    str += "<div class='alert alert-warning'>";
                    str += "<b>" + writer + " : " + content.msg_content +  "</b>";
                    str += "<p>" + date + "</p>";
                    str += "</div></div>";
                    $("#msgArea").append(str);
                }

                // $("#msgArea").append(str);
            });

            //3. send(path, header, message)로 메세지를 보낼 수 있음
            stomp.send('/pub/chat/enter', {}, JSON.stringify({msg_idx: roomId, send_id: username}))
        });

        $("#button-send").on("click", function (e) {
            var msg = document.getElementById("msg");

            console.log(username + ":" + msg.value);
            stomp.send('/pub/chat/message', {}, JSON.stringify({
                msg_idx: roomId,
                msg_content: msg.value,
                send_id: username,
                receive_id: receiveName
            }));
            msg.value = '';
        });
    });
```
	
</details>
	

<details>
<summary> <b>설정</b> </summary>
  
- 라이브러리 설치

```xml
		<dependency>
			<groupId>org.springframework</groupId>
			<artifactId>spring-messaging</artifactId>
			<version>5.2.18.RELEASE</version>
		</dependency>
		<!-- https://mvnrepository.com/artifact/org.webjars/stomp-websocket -->
		<dependency>
			<groupId>org.webjars</groupId>
			<artifactId>stomp-websocket</artifactId>
			<version>2.3.4</version>
		</dependency>
```

</details>
  


<details>
<summary> <b>어려웠던 점</b> </summary>

- 간단한 1:1 대화는 기본 websocket을 활용하였다.
- 여러개의 채팅방이 필요 했으므로, 코드가 복잡해지기 시작했다. (자료구조가 복잡해졌다.)
- 고로, STOMP를 이용해 처음부터 다시 작성해야 했다. (웹소켓을 어느정도 이해한 후라 이해하기 수월했다.)
- 가장 어려운 점은 STOMP 구현 후에 고려해야 할  조건 이었다.(로그를 불러오거나 저장, 조건에 의해 방 생성제한 혹은 생성, 읽음확인 등등)

</details>
	
<details>
<summary> <b>앞으로 해야될 것</b> </summary>

- 코드를 깔끔하게 정리하는 습관을 들이자.
- Controller 와 Service에서 해야할 것들을 잘 구분해야 된다.
- 조건이 어떻게하면 더 간단할지 생각해봐야 한다.
- 읽음표시 기능이 아직 완벽하지 않으며, 채팅방 나가기를 구현해야한다. 
- 웹소켓으로 다중채팅을 구현하려고 했을때 알고리즘 학습의 중요성을 알게되었다.

</details>

	
---	
	
	
	
### 4.6. 여행 플랜 게시판 CRUD

- UI, UX 작업 진행중(2023.05.04 기준)

![image](https://user-images.githubusercontent.com/120711406/236082373-9bc28945-930b-4bb1-b97a-9467dd3178e7.png)

![image](https://user-images.githubusercontent.com/120711406/236082809-f12ec646-4e5b-446a-91ba-bb5df6a3a25e.png)

![image](https://user-images.githubusercontent.com/120711406/236082905-87c569c5-4f98-42d5-ba1c-d29a4bfcb3eb.png)

![image](https://user-images.githubusercontent.com/120711406/236082985-f3c54b13-b09a-483e-af48-94313a5830ee.png)


<details>
<summary> <b> 기능 설명 </b> </summary>
	
- 이미지 리스트 형식의 게시판
- 일반 유저는 플래너가 작성한 여행 플랜을 구매할 수 있음.
- 일반 유저는 여행 플랜을 카트에 담을 수 있음.
- 일반 유저는 해당 플래너와 채팅을 할 수 있음.
- 플래너는 플랜을 생성할 수 있음.
- 플래너는 유저의 문의사항을 채팅을 통해 처리할 수 있음.
- 플래너는 본인이 올린 플랜을 수정/삭제 할 수 있음.
- 삭제는 DB에서 영구 삭제되지 않고 해당 컬럼만 변경됨.

	
</details>

<details>
<summary> <b> Controller </b> </summary>

- 작성, 리스트, 수정, 삭제 구현
- 권한, 페이징, S3업로드 공통클래스 사용

```java
@Controller
@RequestMapping("plan/*")
public class PlanController {

    final PlanService planService;
    final UserService userService;
    final S3FileUploadService s3FileUploadService;

    //생성자 의존성 주입
    public PlanController(PlanService planService, UserService userService, S3FileUploadService s3FileUploadService) {
        this.planService = planService;
        this.userService = userService;
        this.s3FileUploadService = s3FileUploadService;
    }

    // 작성 get
    @Auth(role = Auth.Role.PLANNER)
    @GetMapping("create")
    public String planGet() {
        return "plan/plan_create";
    }

    // 작성 post
    @PostMapping("create")
    public String planPut(PlanDTO planDTO, ImgDTO imgDTO, HttpSession httpSession,
                          @RequestParam("files[]") List<MultipartFile> multipartFile) throws IOException {
        String user = (String) httpSession.getAttribute("user_id");
        planDTO.setUser_id(user);
        int plan_idx = planService.planCreate(planDTO); // 게시글 생성
        if(plan_idx!=0){ // 이미지 파일 생성
            if(multipartFile !=null || !multipartFile.isEmpty()){ // 이미지 파일 있으면
                List<String> imgUrlList = s3FileUploadService.upload(multipartFile); // 서버에 이미지 파일 저장 후 URL값 List에 담기
                planDTO.setPlan_idx(plan_idx); // 게시글 인덱스 set
                planDTO.setP_img(imgUrlList); // 이미지 url set
                boolean success = this.planService.planImgCreate(planDTO); // 이미지 저장 성공
                if(success){
                    return "redirect:/plan/list";
                }
            }
        }
        return "/plan/plan_create";
    }

    // 리스트 겟
    @GetMapping("list")
    public ModelAndView mv(ModelAndView modelAndView, Criteria cri, PlanDTO planDTO) {
        List<PlanDTO> originalList = planService.imgList(planDTO);
        List<PlanDTO> newList = new ArrayList<>(); // 인덱스+첫번째 이미지 값만 있는 dto 담을 List
        for(PlanDTO dto : originalList){
            List<String> planImgList = dto.getP_img(); // 이미지만 List에 담기
            if(planImgList != null && !planImgList.isEmpty()){ //  이미지가 있는 경우
                String firstImg = planImgList.get(0); // 첫번째 이미지 변수에 담기
                PlanDTO newDto = new PlanDTO(); // 인덱스+첫번째 이미지 값 담을 dto
                newDto.setPlan_idx(dto.getPlan_idx()); // 인덱스 담기
                newDto.setP_img(Collections.singletonList(firstImg)); // 첫번째 이미지 담기
                newList.add(newDto);
            }
        }
        System.out.println("newList첫번째이미지 : " + newList.get(0).getP_img());
        System.out.println("data : " + planService.list(cri));
        modelAndView.addObject("imgList", newList); // 게시글 이미지 데이터
        modelAndView.addObject("paging", planService.paging(cri)); // 페이징
        modelAndView.addObject("data", planService.list(cri)); // 게시글 데이터
        modelAndView.setViewName("plan/plan_list");
        return modelAndView;
    }

    // 디테일
    @GetMapping("list/{plan_idx}")
    public ModelAndView planDetail(ModelAndView modelAndView, @PathVariable("plan_idx") int plan_idx) {
        modelAndView.addObject("data", planService.detail(plan_idx));
        modelAndView.setViewName("plan/plan_detail");
        return modelAndView;
    }

    // 수정 겟
    @GetMapping("list/edit")
    public ModelAndView planEdit(ModelAndView modelAndView, HttpSession httpSession,
                                 @RequestParam("idx") int plan_idx, @RequestParam("auth") String user_id) {
        String user = (String) httpSession.getAttribute("user_id");
        if (user.equals(user_id)) {
            modelAndView.addObject("data", planService.detail(plan_idx));
            modelAndView.setViewName("plan/plan_edit");
        } else {
            modelAndView.setViewName("/plan/plan_list");
        }
        return modelAndView;
    }

    // 수정 포스트
    @PostMapping("list/edit")
    public String planEditPut(PlanDTO planDTO, HttpSession httpSession,
                              @RequestParam("idx") int plan_idx, @RequestParam("auth") String user_id,
                              @RequestParam("files[]") List<MultipartFile> multipartFiles) {
        String user = (String) httpSession.getAttribute("user_id");
        if (user.equals(user_id)) {
            planDTO.setPlan_idx(plan_idx);
            planService.planEdit(planDTO); // 게시글 업데이트 (이미지 제외)
            for (String fileName : planService.detail(plan_idx).getP_img()) { // list에 담겨있는 URL값 가져오기
                s3FileUploadService.deleteFromS3(fileName); // s3서버 이미지 파일 삭제
            }
            boolean success = planService.planImgDelete(planDTO); // 기존 이미지 파일 삭제
            if(success){  // 이미지 업데이트
                try {
                    if (multipartFiles != null || !multipartFiles.isEmpty()) {
                        List<String> imgList = s3FileUploadService.upload(multipartFiles);
                        planDTO.setP_img(imgList);
                        planDTO.setPlan_idx(plan_idx);
                        this.planService.planImgUpdate(planDTO);
                    }
                } catch (IOException e){
                    throw new RuntimeException(e);
                }
            }
            return "redirect:/plan/list";
        } else {
            return "redirect:/plan/list/edit";
        }
    }
    // 삭제
    @PostMapping("list/delete")
    public String planDelete(int plan_idx, PlanDTO planDTO) {
        planDTO.setPlan_idx(plan_idx);
        planService.planDelete(plan_idx); // 게시글 삭제
        for (String fileName : planService.detail(plan_idx).getP_img()) {
            s3FileUploadService.deleteFromS3(fileName); // s3서버 이미지 파일 삭제
        }
        planService.planImgDelete(planDTO); // 이미지 삭제
        return "redirect:/plan/list";
    }
}
```

</details>


<details>
<summary> <b>Service</b> </summary>

```java
@Service
public class PlanServiceImpl implements PlanService {

    final
    PlanDAO planDAO;

    public PlanServiceImpl(PlanDAO planDAO) {
        this.planDAO = planDAO;
    }

    // 플랜작성
    @Override
    public int planCreate(PlanDTO planDTO) {
        int affectRowCnt = this.planDAO.create(planDTO);
        if(affectRowCnt!=0){
            return planDTO.getPlan_idx();
        }
        return 0;
    }

    //플랜 디테일
    @Override
    public PlanDTO detail(int plan_idx) {
        return planDAO.detail(plan_idx);
    }

    //이미지 업로드
    @Override
    public boolean planImgCreate(PlanDTO planDTO) {
        int affectRowCnt = this.planDAO.planImgCreate(planDTO);
        if(affectRowCnt!=0){
            return true;
        }
        return false;
    }

    //플랜 수정
    @Override
    public void planEdit(PlanDTO planDTO) {
        planDAO.edit(planDTO);
    }

    @Override
    public boolean planImgDelete(PlanDTO planDTO) {
        int affectRowCnt = this.planDAO.planImgDelete(planDTO);
        if(affectRowCnt !=0){
            return true;
        }
        return false;
    }

    @Override
    public void planImgUpdate(PlanDTO planDTO) {
        this.planDAO.planImgUpdate(planDTO);
    }

    //플랜 리스트
    @Override
    public List<PlanDTO> list(Criteria cri) {
        return planDAO.list(cri);
    }

    // paging처리 - 04.18 김범수
    @Override
    public PagingDTO paging(Criteria cri) {
        PagingDTO paging = new PagingDTO();
        paging.setCri(cri);
        paging.setTotalCount(planDAO.totalConut(cri));
        return paging;
    }

    @Override
    public List<PlanDTO> imgList(PlanDTO planDTO) {

        return this.planDAO.ImgList(planDTO);
    }

    //플랜 삭제
    @Override
    public void planDelete(int planIdx) {
        planDAO.delete(planIdx);
    }

}
```

</details>

<details>
<summary> <b>SQL</b> </summary>

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="plan">

    <!--플랜만들기-->
    <insert id="create" parameterType="com.goott.pj3.plan.dto.PlanDTO" useGeneratedKeys="true" keyProperty="plan_idx">
        INSERT INTO plan(user_id, start_date, end_date, price, plan_title, plan_detail)
        VALUES (#{user_id}, #{start_date}, #{end_date}, #{price}, #{plan_title}, #{plan_detail})
    </insert>

    <insert id="createImg" parameterType="com.goott.pj3.plan.dto.PlanDTO">
        insert into plan_img(plan_idx, p_img)
             values
                   <foreach collection="p_img" item="img" separator=",">
                       (#{plan_idx}, #{img})
                   </foreach>
    </insert>

    <!--플랜리스트-->
    <select id="list" resultType="com.goott.pj3.plan.dto.PlanDTO">
        SELECT plan_idx, plan_title, price, user_id
        FROM plan
        WHERE p_del_yn = 'N'
        AND
        <if test="option == 'user_id'">user_id like CONCAT('%',#{keyword},'%')</if>
        <if test="option == 'title'">plan_title like CONCAT('%',#{keyword},'%')</if>
        <if test="option == null or option == ''">1=1</if>
        ORDER BY plan_idx DESC
        LIMIT #{pageStart}, #{perPageNum}
    </select>

    <select id="imgList" resultMap="planListResultMap">
           select i.p_img, i.plan_idx
             from plan_img i left join plan p
               on i.plan_idx = p.plan_idx
            where p.p_del_yn='n'
         order by p.plan_idx desc
    </select>

    <resultMap id="planListResultMap" type="com.goott.pj3.plan.dto.PlanDTO">
        <id property="plan_idx" column="plan_idx"/>
        <collection property="p_img" column="p_img" javaType="List" ofType="String">
            <result column="p_img"/>
        </collection>
    </resultMap>

    <!--페이징을 위한 카운트-->
    <select id="totalCount" resultType="int">
        SELECT count(plan_idx)
        FROM plan
        WHERE p_del_yn = 'N'
        AND
        <if test="option == 'user_id'">user_id like CONCAT('%',#{keyword},'%')</if>
        <if test="option == 'title'">plan_title like CONCAT('%',#{keyword},'%')</if>
        <if test="option == null or option == ''">1=1</if>
    </select>

    <!--플랜상세-->
    <select id="detail" resultMap="planResultMap">
        SELECT p.plan_title, p.plan_detail, p.user_id,
               p.start_date, p.end_date, p.price, p.plan_idx, i.p_img_idx, i.p_img
         FROM plan p left join plan_img i
           on p.plan_idx = i.plan_idx
        WHERE p.plan_idx = #{plan_idx}
          and p.p_del_yn='n'
    </select>

    <resultMap id="planResultMap" type="com.goott.pj3.plan.dto.PlanDTO">
        <id property="plan_idx" column="plan_idx"/>
        <result property="plan_title" column="plan_title"/>
        <result property="plan_detail" column="plan_detail"/>
        <result property="user_id" column="user_id"/>
        <result property="start_date" column="start_date"/>
        <result property="end_date" column="end_date"/>
        <result property="price" column="price"/>
        <collection property="p_img_idx" column="p_img_idx" javaType="List" ofType="String">
            <result column="p_img_idx"/>
        </collection>
        <collection property="p_img" column="p_img" javaType="List" ofType="String">
            <result column="p_img"/>
        </collection>
    </resultMap>

    <!--플랜수정-->
    <update id="edit">
        UPDATE plan
        SET plan_title = #{plan_title},
            plan_detail=#{plan_detail},
            start_date=#{start_date},
            end_date=#{end_date},
            price=#{price}
        WHERE plan_idx = #{plan_idx}
    </update>

    <delete id="planImgDelete">
        delete
          from plan_img
         where plan_idx=#{plan_idx}
    </delete>

    <insert id="planImgUpdate" parameterType="com.goott.pj3.plan.dto.PlanDTO">
        insert into plan_img(plan_idx, p_img)
             values
                   <foreach collection="p_img" item="img" separator=",">
                       (#{plan_idx}, #{img})
                   </foreach>
    </insert>
    <!--.플랜 수정 끝-->

    <!--플랜삭제(DB삭제는 안함)-->
    <update id="delete">
        UPDATE plan
        SET p_del_yn = 'Y'
        WHERE plan_idx = #{plan_idx}
    </update>
    <!--플랜결제-->
    <insert id="pay" parameterType="com.goott.pj3.plan.dto.PayDTO">
        INSERT INTO pay(user_id, plan_idx, imp_uid, merchant_id)
        VALUES (#{buyer_id},#{plan_idx},#{imp_uid},#{merchant_uid})
    </insert>
    <!--결제된 플랜 판매+1-->
    <update id="count">
        UPDATE plan
        SET sale_count = (sale_count+1)
        WHERE plan_idx=#{plan_idx}
    </update>
    <!--결제된 플랜 플래너성공 +1-->
    <update id="success">
        UPDATE user
        SET success_count =(success_count+1)
        WHERE user_id = #{planner_id}
    </update>
</mapper>
```
  

</details>

<details>
<summary> <b>JSP & JavaScript</b> </summary>

- 플랜 디테일

```jsp
<input class="session" type="hidden" value="${sessionScope.user_id}">
<input class="plan_idx" type="hidden" value="${data.plan_idx}">

<label for="title">제목</label>
<input id="title" type="text" value="${data.plan_title}">

<label for="price">가격</label>
<input id="price" type="text" value="${data.price}">

<label for="detail">설명</label>
<input id="detail" type="text" value="${data.plan_detail}">

<p> 이미지: </p>
<c:forEach var="img" items="${data.p_img}">
    <img src="${img}" width="200" height="200" style="border: 1px solid blue;">
</c:forEach>

<%--<c:set var = "date_count" value = "${data.end_date - data.start_date}"/>--%>
<%--<c:out value="${date_count}"/>--%>

<p>기간 : </p>
<p>시작날짜 : ${data.start_date}</p>
<p>종료날짜 : ${data.end_date}</p>
<label for="planner">플래너</label>
<input id="planner" type="text" value="${data.user_id}">

<form action="/chat/room" method="post">
    <p>폼태그 안-> 추후 hidden</p>
    <input type="hidden" name="name" id="name" class="form-control" value="">
    <input type="hidden" name="send_id" id="send_id" class="form-control" value="${sessionScope.user_id}">
    <input type="hidden" name="receive_id" id="receive_id" class="form-control" value="${data.user_id}">
    <c:if test="${sessionScope.auth == 'auth_c'}">
    <button type="submit" class="btn btn-secondary">플래너에게 메세지 보내기</button>
    </c:if>
</form>

<c:if test="${data.user_id == sessionScope.user_id}">
    <button type="button" onclick="location.href='edit?idx=${data.plan_idx}&auth=${data.user_id}'">수정</button>
    <button data-id="${data.plan_idx}" id="delete">삭제</button>
</c:if>
<button id="cart" type="button" onclick="addCart()">카트담기</button>
<button type="button" onclick="kakao()">결제</button>
```

- 플랜 

```javascript
    async function previewFile() {
        var preview = document.getElementById("preview"); // 미리보기 띄울 div
        var files = document.getElementById('file-input').files; // img 파일들
        var cnt = 0; // 이미지 갯수
        preview.innerHTML = ''; // 미리보기 초기화

        for (const file of files) {  // 반복문 한번 반복 때마다 이미지 1개씩 view
            await new Promise((resolve, reject) => {
                var reader = new FileReader(); // FileReader 객체를 생성
                reader.onload = function() { // 파일 로드가 성공시 호출 될 함수
                    var img = document.createElement("img"); // img 생성
                    img.src = reader.result; // 로드된 파일을 img 요소의 src에 할당
                    img.onload = () => { // 이미지 로드가 완료되면 이 함수가 호출
                        preview.appendChild(img); // preview 요소의 자식 노드로 img 추가
                        resolve(); // 결과 호출
                        cnt++ // 이미지 갯수 더하기
                        if(cnt == files.length){
                            $('#upload-btn').prop('disabled', false); // 이미지 파일 올리면 저장버튼 활성화
                        }
                        else if(cnt != files.length){
                            $('#upload-btn').prop('disabled', true); // 이미지 파일 취소 할 경우 다시 비활성화
                        }
                    }
                };
                reader.onerror = function() {
                    reject(new Error('파일 로드 실패'));
                };
                reader.readAsDataURL(file);
            });
        }
    }
    /**
     * 이미지 업로드 조건
     * @type {RegExp}
     */
    let regex = new RegExp("(.*?)\.(jpg|png)$");         // jpg,png 파일만 허용
    let maxSize = 41943040;                              // file 제한 용량 40MB

    $("input[type='file']").on("change", function(e){
        let fileInput = document.querySelector("#fileItem");
        let fileList = fileInput.files;
        let fileObj = fileList[0];

        if(!fileCheck(fileObj.name, fileObj.size)) return false;
        alert("통과")
    });

    // 이미지 체크 로직
    function fileCheck(fileName, fileSize){
        if(fileSize >= maxSize){
            alert("파일 사이즈 초과 : 최대 40MB");
            return false;
        }
        if(!regex.test(fileName)){
            alert("해당 종류의 파일은 업로드할 수 없습니다. 업로드 가능한 file : jpg, png");
            return false;
        }
    }
```

</details>
	  
<details>
<summary> <b>어려웠던 점</b> </summary>

- 기능이 늘어나고 작업자가 늘어날수록 서로의 스타일이 달라서 복잡해졌다. 
- mybatis에서 결과값을 return받을때 null 처리와 return type 때문에 고생했음.
  

</details>
  
<details>
<summary> <b>앞으로 해야될 것</b> </summary>

- 서비스와 컨트롤러의 분리를 더 철저히 하는 습관을 가져보자.
- 협업시 조금더 철저하게 디폴트 기준을 잘 정해놓고 서로 소통하자.

</details>


---


### 4.7. 카트

- UI, UX 구현중(2023.05.03 기준)

![image](https://user-images.githubusercontent.com/120711406/236091379-758e9633-fa31-4bfb-b13d-0c6985ece2fc.png)


<details>
<summary> <b> 기능설명 </b> </summary>
	
- 유저는 구매하고자하는 여행 플랜을 카트에 담을 수 있다.
- 카트는 체크박스를 이용해 다중 삭제를 할 수 있다.
- 플랜은 중복되어 담기지 않는다.

</details>

<details>
<summary> <b>Controller</b> </summary>

```java
@Controller
public class CartController {
    final
    CartService cartService;

    public CartController(CartService cartService) {
        this.cartService = cartService;
    }

    // 카트추가
    @PostMapping(value = "/addcart", consumes = "application/json")
    @ResponseBody
    public Map<String, Object> planCart(@RequestBody PlanDTO planDTO) {
        cartService.addCart(planDTO);
        Map<String, Object> map = new HashMap<>();
        map.put("cart", "카트담기");
        return map;
    }

    //카트 보여주기 데모
    @GetMapping("cart")
    public ModelAndView cart(ModelAndView mv, PlanDTO planDTO, HttpSession httpSession) {
        String user = (String) httpSession.getAttribute("user_id");
        planDTO.setUser_id(user);
        mv.addObject("cart", cartService.getCart(planDTO));
        mv.setViewName("cart/cart_demo");
        return mv;
    }

    //카트 삭제
    @DeleteMapping("cart/delete")
    public String delete(@RequestParam("delList") List<Integer> list) {
        for (Integer plan_idx : list) cartService.deleteCart(plan_idx);
        return "redirect:/cart";
    }
}

```

</details>
	
<details>
<summary> <b>Service</b> </summary>

```java
@Service
public class CartServiceImpl implements CartService {
    final
    CartDAO cartDAO;

    public CartServiceImpl(CartDAO cartDAO) {
        this.cartDAO = cartDAO;
    }

    //카트추가
    @Override
    public void addCart(PlanDTO planDTO) {
        cartDAO.addCart(planDTO);
    }

    //카트불러오기
    @Override
    public List<PlanDTO> getCart(PlanDTO planDTO) {
        return cartDAO.getCart(planDTO);
    }

    //카트삭제
    @Override
    public void deleteCart(int planIdx) {
        cartDAO.deleteCart(planIdx);

    }
}

```
	
</details>
	
<details>
<summary> <b>SQL</b> </summary>
	
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="cart">
    <!--카트추가-->
    <insert id="add" parameterType="com.goott.pj3.plan.dto.PayDTO">
        INSERT INTO plan_cart(user_id, plan_idx)
        SELECT #{user_id}, #{plan_idx}
        FROM dual
        WHERE not exists(select * from plan_cart where user_id = #{user_id} And plan_idx = #{plan_idx})
    </insert>
    <!--카트가져오기-->
    <select id="get" resultType="com.goott.pj3.plan.dto.PlanDTO">
        SELECT plan_title, plan_idx, p_del_yn
        FROM plan
        WHERE plan_idx in (select plan_idx
                           from plan_cart
                           where user_id = #{user_id})
    </select>
    <!--카트삭제-->
    <delete id="delete">
        DELETE
        FROM plan_cart
        WHERE plan_idx = #{plan_idx}
    </delete>
    <!--결제시 카트빼기-->
    <delete id="subCart">
        DELETE
        FROM plan_cart
        WHERE user_id = #{buyer_id}
          AND plan_idx = #{plan_idx}
    </delete>
</mapper>
```
	
</details>
	
<details>
<summary> <b>jsp & JavaScript</b> </summary>
	
- 카트담기

```javascript
    function addCart() {
        let cart = {
            plan_idx: plan_idx,
            user_id: buyer
        };
        $.ajax({
            type: 'Post',
            url: '/addcart',
            dataType: 'json',
            contentType: 'application/json; charset=utf-8',
            data: JSON.stringify(cart)
        }).done(function (rsp) {
            if (rsp.cart === '카트담기') {
                alert('카트담기성공')
            } else {
                alert('카트담기실패');
            }

        }).fail(function (error) {
            alert('에이젝스 실패')
        })
    }

```
	
- 카트 페이지

```jsp
<form name="delete" action="cart/delete" method="POST">
    <input type="hidden" name="_method" value="delete"/>
    <c:if test="${not empty cart}">
        <tr>
            <c:forEach var="item" items="${cart}">
                <c:if test="${item.p_del_yn eq 'N'}">
                    <td><a href="plan/list/${item.plan_idx}">${item.plan_title}</a></td>
                    <td>${item.plan_idx}</td>
                    <input name="delList" type="checkbox" value="${item.plan_idx}"/>
                    <br>
                </c:if>
            </c:forEach>
            <button type="submit">삭제</button>
```
	
</details>
	
	
---
	

	
### 4.7. 그 외
	
- Restfull한 라우팅을 위해서 httpMethodFilter를 사용.

![image](https://user-images.githubusercontent.com/120711406/236096218-a127c7eb-bff5-46e0-a053-21baa9f237ed.png)
	
- 에러 핸들링

![image](https://user-images.githubusercontent.com/120711406/236096287-01f1f272-41a0-4b99-8cd7-e8349c1f0c3a.png)

	
---
	
	

## 5. 프로젝트 전반적인 후기
	
<details>
<summary> <b>어려웠던 점</b> </summary>
	
- 직전 프로젝트가 Nodejs였기 때문에, 다시 Java에 적응해야 했음.(사실 Java의 매력을 더 느끼게됨)
- 3번째 프로젝트임에도 불구하고 협업은 항상 어려움/(기본적으로 정해놓고 가야될 것들이 더 상세해야됨)
- DB 설계의 영역은 미래를 내다보는 듯한 영역이라고 느껴짐.(경험이 중요한것 같음)
- Spring Legacy를 사용하여 외부 라이브러리를 쓸때엔 최신정보를 찾기 매우 힘들었음.
- UI 작업자와 소통이 부족했던것 같음. (동상이몽)
	
</details>

<details>
<summary> <b>발전된 점</b> </summary>

- 구조와 흐름이 보임
- 커리큘럼에 없었던 Nodejs로 2차프로젝트를 해본 결과, 새로운 라이브러리나 기술에 대해 두려움이 사라짐.
- 주석을 전보다 잘 쓰게됨. (더 명확하고 간결하게 쓰도록 발전해야함.)
- AWS와 리눅스, ci/cd 등 더 넓은 시야를 갖게 됨
- 에러 해결능력이 많이 향상됨 (구조와 흐름이 이제 보이기 떄문인것 같음)
	
</details>
	
<details>
<summary> <b>앞으로 해야할 것</b> </summary>

- Github action 연동 ci/cd 배포까지
- Ubuntu
- 프로젝트 전반적인 코드리뷰와 리팩토링
- Spring Test 과정 학습
- 
	
</details>


