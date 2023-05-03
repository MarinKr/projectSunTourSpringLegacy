# 💡여행 플랜 서비스 플랫폼(GOOTTFLEX)



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

</details>
  
<details>
<summary> <b>앞으로 해야될 것</b> </summary>
	
- AWS RDS 테스트중 추가 결제가 되었음. 학습이 더 필요함.
- 깃허브 액션과 S3 EC2 연계로 CICD구현(진행중)
- EC2 학습 진행중 리눅스 학습의 필요성을 느낌.
	
</details>


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
  
- 엔드포인트와 publish subscribe 값 설정
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
	
</detail>
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
  <summary> <b>네비바 이미지 출력</b> </summary>

  ```java
	// 네비바 프로필 이미지 호출
	@RequestMapping(value={"navbarImg1", "navbarImg2"}, produces = MediaType.APPLICATION_JSON_VALUE)
	public ResponseEntity<String> navbarImg(HttpSession session, UserDto dto, 
					HttpServletRequest request) {
		String requestUrl = request.getRequestURL().toString(); 
		// 매핑값마다 실행되는 코드를 다르게 하기 위해 HttpServletRequest 객체로 매핑 조정
		if(requestUrl.contains("navbarImg1")) { // 네비바1에서의 프로필 로딩
			String id =(String) session.getAttribute("user_id").toString();
			String img = userService.navbarImg(id); 
			// 아이디 정보를 통해서 해당 사용자의 프로필 경로를 받아옴
			// 이미지 값을 ResponseEntity로 변환
			// -> 프로필 이미지 절대 경로와 HTTP 상태 코드를 객체에 넣음
			ResponseEntity<String> result1 = new ResponseEntity<String>(img, HttpStatus.OK); 
			return result1;
		}
		else { // 네비바2에서의 프로필 로딩
			String id =(String) session.getAttribute("user_id").toString();
			String img = userService.navbarImg(id);
			ResponseEntity<String> result2 = new ResponseEntity<String>(img, HttpStatus.OK);
			return result2;
		}
	}

  ```

  </details>

  <details>
  <summary> <b>프로필 이미지 등록/변경</b> </summary>

   ```java

	  // 프로필 등록
	  @RequestMapping(value="upload", produces = MediaType.APPLICATION_JSON_VALUE)
	  public ResponseEntity<ImgDto> test(MultipartFile uploadFile, ImgDto dto, HttpSession session, ModelAndView mv) { 
	      // 세션에 저장된 사용자의 아이디 변수 id에 저장
	      String id = (String) session.getAttribute("user_id"); 

	      // 이미지 파일 체크
	      File checkfile = new File(uploadFile.getOriginalFilename()); // 파일명을 불러옴
	      String type = null;
	      try {
		  type = Files.probeContentType(checkfile.toPath()); 
		  // 해당 파일의 확장자를 불러옴, 확장자가 없으면 type은 null값을 반환
	      } catch (IOException e) {
		  e.printStackTrace();
	      }
	      // Date 객체로 날짜 경로 만들기 
	      // - 하나의 파일에 파일이 많아지면 데이터 관리에 부담이 생김		
	      SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd-"); 
	      // 뒤에 '-'을 더 붙인 이유 : 날짜와 파일명을 구분하기 위함
	      Date date = new Date();
	      String str = sdf.format(date); // yyyy-MM-dd 형식으로 날짜가 들어감
	      // '-'를 separator(파일 구분자)로 나눠 놓음 
	      // -> 2023 - 02 - 24 형식으로 폴더를 만들기 위함
	      String datePath = str.replace("-", File.separator);  

	      String uploadFolder = "C:\\upload\\"; // 처음 경로 설정
	      File uploadPath = new File(uploadFolder, datePath); 
	      // 파일 생성 클래스 -> 파일 객체 생성

	      if(uploadPath.exists() == false) { // 해당 경로에 파일이 없으면 파일 생성
		      uploadPath.mkdirs();
	      }

	      // 파일 이름
	      String uploadFileName = uploadFile.getOriginalFilename();
	      dto.setFileName(uploadFileName);
	      dto.setUploadPath(datePath);

	      // uuid적용한 파일 이름
	      String uuid = UUID.randomUUID().toString(); 
	      // uuid 생성, 파일을 구분하는 키값을 생성하기 위함
	      uploadFileName = uuid + "_" + uploadFileName;
	      dto.setUuid(uuid);

	      String saveFilestr = uploadPath + uploadFileName;
	      File saveFile = new File(uploadPath, uploadFileName); 

	      try {
		  uploadFile.transferTo(saveFile); // saveFile을 저장
		  dto.setSaveFileStr(saveFilestr);
		  dto.setId(id);
		  userService.img_update(dto); 
		  // 해당 사용자의 아이디에 프로필 이미지 경로를 등록하기 위함
	      } catch (IOException e) {
		  e.printStackTrace();
	      }

	      // ResponseEntity 객체로 HTTP 상태 코드와 이미지 경로를 저장
	      ResponseEntity<ImgDto> result = new ResponseEntity<ImgDto>(dto, 
	      HttpStatus.OK); 

	      return result;		
	  }

	  // 이미지 출력 메소드
	  @RequestMapping(value = "display")
	  public ResponseEntity<byte[]> display(String fileName) 
			  throws FileNotFoundException { // 이미지 파일을 바이트 배열로 받아옴

	      File file = new File(fileName);
	      if (!file.exists() || !file.canRead()) { // 파일이 없는 경우
		  throw new FileNotFoundException("The file '" + fileName + "' 을 찾을수 없습니다.");
	      }
	      ResponseEntity<byte[]> result = null; // ResponseEntity 객체 초기화

	      try {
		  HttpHeaders header = new HttpHeaders(); // HttpHeaders 객체 생성
		  header.add("Content-type", Files.probeContentType(file.toPath())); 
		  // 헤더 객체에 Content-type을 파일 확장자로 설정 
		  // ResponseEntity 객체에 이미지 바이트 배열화된 파일 복사한 것과  
		  // HttpHeaders 객체, HTTP 상태 코드를 담음
		  result = new ResponseEntity<>(FileCopyUtils.copyToByteArray(file), header, 
		  HttpStatus.OK);
	      } catch (IOException e) {
		  e.printStackTrace();
	      }
	      return result;
	  }

	  // 프로필 상시 
	  @RequestMapping(value="onload", produces = MediaType.APPLICATION_JSON_VALUE)
	  public ResponseEntity<String> onload(String uploadFile) { 
	      // 절대 경로로 된 이미지를 HTTP 상태 코드와 함께  ResponseEntity 객체에 담음
	      ResponseEntity<String> result = new ResponseEntity<String>(uploadFile, 
	      HttpStatus.OK);
	      return result;		
	  }
   ```

  </details>


</details>
	  
<details>
<summary> <b>DTO</b> </summary>

- 테이블에 들어 있는 정보를 미리 변수로 생성하고 getter/setter를 설정한 파일입니다.
 
```java 

package com.test.test1.mypage.dto;

public class ImgDto {

	/* 경로 */
	private String uploadPath;
	
	/* uuid */
	private String uuid;
	
	/* 파일 이름 */
	private String fileName;
	
	private String saveFilestr;
	
	private String id;
	

	public String getUploadPath() {
		return uploadPath;
	}

	public void setUploadPath(String uploadPath) {
		this.uploadPath = uploadPath;
	}

	public String getUuid() {
		return uuid;
	}

	public void setUuid(String uuid) {
		this.uuid = uuid;
	}

	public String getFileName() {
		return fileName;
	}

	public void setFileName(String fileName) {
		this.fileName = fileName;
	}

	public String getSaveFile() {
		return saveFilestr;
	}

	public void setSaveFileStr(String saveFilestr) {
		this.saveFilestr = saveFilestr;
	}

	public String getId() {
		return id;
	}

	public void setId(String id) {
		this.id = id;
	}

	@Override
	public String toString() {
		return "ImagDto [uploadPath=" + uploadPath 
			+ ", uuid=" + uuid 
			+ ", fileName=" + fileName 
			+ ", saveFilestr=" + saveFilestr 
			+ ", id=" + id 
			+"]";
	}
}
  

```
</details>

<details>
<summary> <b>Service / ServiceImpl</b> </summary>
  

- Controller로 받은 데이터를 DAO로 전달합니다.


```java 
// 네비바 이미지 출력 기능 Service / ServiceImpl
// Service
public interface UserService {
	String navbarImg(String id);

}

// ServiceImpl
@Service
public class UserServiceImpl implements UserService{
	
	@Inject
	UserDao userDao;

	@Override
	public String navbarImg(String id) {
		return userDao.navbarImg(id);
	}
}

```
  
```java 
// 프로필 이미지 등록/변경 기능 Service / ServiceImpl
// Service
public interface UserService {
	void img_update(ImgDto dto);

}

// ServiceImpl
@Service
public class UserServiceImpl implements UserService{
	
	@Inject
	UserDao userDao;

	@Override
	public void img_update(ImgDto dto) {
		userDao.img_update(dto);
	}
}

```
</details>
	  
<details>
<summary> <b>DAO</b> </summary>
  

- Service에서 전달 받은 데이터를 XML로 전달합니다.
  


```java 
// 네비바 이미지 출력 기능 구현 DAO
@Repository
public class UserDao {
	
	@Inject
	SqlSessionTemplate sqlSessionTemplate;
	
	public String navbarImg(String id) {
		return sqlSessionTemplate.selectOne("user.navbarImg" , id);
	}
}
  
```
  
```java 
// 프로필 이미지 등록/변경 기능 구현 DAO
@Repository
public class UserDao {
	
	@Inject
	SqlSessionTemplate sqlSessionTemplate;
	
	public void img_update(ImgDto dto) {
		sqlSessionTemplate.selectOne("user.img_update" , dto);
	}
}
  
```
</details>
  
<details>
<summary> <b>XML</b> </summary>
  

- DAO에서 전달 받은 데이터로 쿼리문을 통해 프로필 이미지 경로를 등록하거나 리턴합니다.

  <details>
    <summary> <b>네비바 이미지 출력</b> </summary>

    - 해당 사용자의 프로필 이미지 경로를 아이디 정보를 통해 불러옵니다.
 
    ```xml 
    <!-- 네비바 이미지 출력 기능 xml -->
    <select id="navbarImg" resultType="String">
            <![CDATA[
            select IMG
              from USER
             where ID=#{user_id}
            ]]>
    </select>
    ```
 
  </details>
    <details>
    <summary> <b>프로필 이미지 등록/변경</b> </summary>

    - 이미지 경로를 해당 사용자의 정보에 저장합니다.<br>
    - 변경을 할 때에도 위와 같은 방법으로 저장이 이루어집니다. 
 
    ```xml 
    <!-- 프로필 이미지 등록/변경 기능 xml -->
    <update id="img_update">
      update USER
             set IMG = #{saveFilestr}
         where ID = #{id}
    </update>
    ```
 
  </details>

</details>
</details>
