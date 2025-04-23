# **懒人专用聊天系统笔记 📚✨**

> 由于涉及头像、用户信息，所以直接复制代码不能成功，需要根据个人代码进行修改，以下是本人的关于私聊+群聊的实现代码（仅供参考）

## 系统架构概述

这个聊天系统实现了私聊和群聊功能，采用前后端分离架构，后端使用Spring Boot，前端使用Vue.js + Pinia状态管理，通过WebSocket实现实时通信。

## 后端部分 🏗️

### 数据库构建(id均为主键)⚙️

**chat**

| id   | userId | toUserId | message | sender | time       |
| ---- | ------ | -------- | ------- | ------ | ---------- |
| 1    | 1      | 2        | 你好    | 1      | 2025-04-16 |

**group_information**

| id   | groupName | founderId | time       |
| ---- | --------- | --------- | ---------- |
| 1    | 测试1     | 1         | 2025-04-16 |

**group_message**

| id   | groupId | message | sender | time       |
| ---- | ------- | ------- | ------ | ---------- |
| 1    | 1       | 你好    | 1      | 2025-04-16 |

**group_user**

| id   | groupId | userId | time       |
| ---- | ------- | ------ | ---------- |
| 1    | 1       | 1      | 2025-04-16 |

### 实体类大家族 👨‍👩‍👧‍👦

1. **Chat** - 私聊消息实体

   - 记录发送者(userId)、接收者(toUserId)、消息内容(message)、发送者标识(sender)和时间戳

   ```
   @Data
   public class Chat {
       private Integer id,userId,toUserId,sender;
       private String message;
       private Date time;
   
       public  Chat(Integer userId,Integer toUserId,String message,Integer sender,Date time)
       {
           this.userId=userId;
           this.toUserId=toUserId;
           this.message=message;
           this.sender=sender;
           this.time=time;
       }
       public  Chat(Integer id,Integer userId,Integer toUserId,String message,Integer sender,Date time)
       {
           this.id=id;
           this.userId=userId;
           this.toUserId=toUserId;
           this.message=message;
           this.sender=sender;
           this.time=time;
       }
   }
   ```

2. **GroupInformation** - 群组信息实体

   - 包含群ID、创建者ID、群名称和创建时间

   ```
   @Data
   public class GroupInformation {
       private Integer id,founderId;
       private String groupName;
       private Date time;
   
       public GroupInformation( String groupName, Integer founderId,Date time) {
           this.founderId = founderId;
           this.groupName = groupName;
           this.time = time;
       }
   }
   ```

3. **GroupMessage** - 群消息实体

   - 记录群ID、发送者、消息内容和时间

   ```
   @Data
   public class GroupMessage {
       private Integer id,groupId,sender;
       private String message;
       private Date time;
   
       public GroupMessage(Integer groupId,String message,Integer sender,Date time){
            this.groupId = groupId;
            this.message = message;
            this.sender = sender;
            this.time = time;
       }
   
       public GroupMessage(Integer id,Integer groupId,String message,Integer sender,Date time){
           this.id = id;
           this.groupId = groupId;
           this.message = message;
           this.sender = sender;
           this.time = time;
       }
   }
   ```

4. **GroupMessageAndUser** - 群消息+用户头像组合实体

   - 方便前端显示消息时附带用户头像

   ```
   @Data
   public class GroupMessageAndUser {
       private GroupMessage groupMessage;
       private String avatar;
   
       public GroupMessageAndUser(GroupMessage groupMessage, String avatar) {
           this.groupMessage = groupMessage;
           this.avatar = avatar;
       }
   }
   ```

5. **GroupUser** - 群成员关系实体

   - 记录用户与群的关联关系

   ```
   @Data
   public class GroupUser {
       private Integer id,groupId,userId;
       private Date time;
   
       public GroupUser(Integer id,Integer groupId,Integer userId,Date time) {
           this.id=id;
           this.groupId=groupId;
           this.userId=userId;
           this.time=time;
       }
       public GroupUser(Integer groupId,Integer userId,Date time) {
           this.groupId=groupId;
           this.userId=userId;
           this.time=time;
       }
   }
   ```

6. **GroupUserAndInformation** - 群成员+群信息组合实体

   - 方便查询用户所在群组的详细信息

     ```
     @Data
     public class GroupUser {
         private Integer id,groupId,userId;
         private Date time;
     
         public GroupUser(Integer id,Integer groupId,Integer userId,Date time) {
             this.id=id;
             this.groupId=groupId;
             this.userId=userId;
             this.time=time;
         }
         public GroupUser(Integer groupId,Integer userId,Date time) {
             this.groupId=groupId;
             this.userId=userId;
             this.time=time;
         }
     }
     ```

### Mapper层 🗃️

- 使用MyBatis注解方式编写SQL

- 主要功能：

  - 私聊消息的增查
  - 群组的创建、加入
  - 群消息的增查
  - 群成员的查询

  ```
  @Mapper
  public interface ChatMapper {
      //添加私聊
      @Insert("INSERT INTO chat(userId,toUserId,message,sender,time) VALUES (#{userId},#{toUserId},#{message},#{sender},#{time})")
      int addNewChat(Chat chat);
  
      //获取私聊信息
      @Select("SELECT *FROM chat WHERE (userId=#{userId} AND toUserId=#{toUserId}) OR (userId=#{toUserId} AND toUserId=#{userId})")
      List<Chat> getChatByUserIdAndToUserId(Integer userId, Integer toUserId);
  
      //创建群聊
      @Insert("INSERT INTO group_information(groupName,founderId,time) VALUES (#{groupName},#{founderId},#{time})")
      @Options(useGeneratedKeys = true, keyProperty = "id", keyColumn = "id")//让数据库生成的主键值能够自动传回到 Java 对象中
      int CreateGroup(GroupInformation groupInformation);
  
      //加入群聊
      @Insert("INSERT INTO group_user(userId,groupId,time) VALUES (#{userId},#{groupId},#{time})")
      int JoinGroup(GroupUser groupUser);
  
      //获取我的群聊
      @Select("SELECT * FROM group_user,group_information WHERE userId=#{userId} AND group_user.groupId=group_information.id")
      List<GroupUserAndInformation> getGroupByUserId(Integer userId);
  
      //发送群聊信息
      @Insert("INSERT INTO group_message(groupId,message,sender,time) VALUES (#{groupId},#{message},#{sender},#{time})")
      int addNewGroupChat(GroupMessage GroupMessage);
  
      //获取群聊信息
      @Select("SELECT *FROM group_message WHERE groupId=#{groupId}")
      List<GroupMessage> getGroupByGroupId(Integer groupId);
  
      //获取群聊中所有用户
      @Select("SELECT * FROM group_user WHERE groupId=#{groupId}")
      List<GroupUser> getGroupUserByGroupId(Integer groupId);
  }
  ```

### Service层 ⚙️

- 业务逻辑处理

- 亮点功能：

  - 创建群组后自动将创建者加入群聊
  - 查询群消息时自动关联用户头像

  ```
  @Service
  public class ChatServiceImpl implements ChatService {
  
      @Resource
      ChatMapper chatMapper;
      @Resource
      UserMapper userMapper;
  
  
      @Override
      public int addNewChat(Chat chat) {
          return chatMapper.addNewChat(chat);
      }
  
      @Override
      public List<Chat> getChatByUserIdAndToUserId(Integer userId, Integer toUserId) {
          return chatMapper.getChatByUserIdAndToUserId(userId, toUserId);
      }
  
      @Override
      public int CreateGroup(GroupInformation groupInformation) {
          chatMapper.CreateGroup(groupInformation);
          Integer id = groupInformation.getId();
          if (id == null) {
              throw new RuntimeException("创建群组失败，未能获取到群组ID");
          }
          return id;
      }
  
      @Override
      public int JoinGroup(GroupUser groupUser) {
          return chatMapper.JoinGroup(groupUser);
      }
  
      @Override
      public List<GroupUserAndInformation> getGroupByUserId(Integer userId) {
          return  chatMapper.getGroupByUserId(userId);
      }
  
      @Override
      public int addNewGroupChat(GroupMessage groupMessage) {
          return chatMapper.addNewGroupChat(groupMessage);
      }
  
      @Override
      public List<GroupMessageAndUser> getGroupByGroupId(Integer groupId) {
          List<GroupMessageAndUser> groupMessageAndUsers = new ArrayList<>();
          List<GroupMessage> groupMessage = chatMapper.getGroupByGroupId(groupId);
          for (GroupMessage message : groupMessage) {
              groupMessageAndUsers.add(new GroupMessageAndUser(message,userMapper.getAvatorById(message.getSender())));
          }
          return groupMessageAndUsers;
      }
  
      @Override
      public List<GroupUser> getGroupUserByGroupId(Integer groupId) {
          return chatMapper.getGroupUserByGroupId(groupId);
      }
  }
  ```

### Controller层 🎮

- 提供RESTful API

- 接口分类：

  - 私聊相关：发送/获取私聊消息
  - 群聊相关：创建/加入群组、发送/获取群消息

  ```
  @RestController
  @RequestMapping("api/chat")
  public class ChatController {
      @Resource
      ChatService chatService;
  
      @PostMapping("/PrivateChat")
      public RestBean<String> PrivateChat(HttpServletRequest request,
                                          @RequestParam("toUserId") int toUserId,
                                          @RequestParam("message") String message)
      {
          Integer userId=(Integer)request.getAttribute("id");
          Chat chat =new Chat(userId,toUserId,message,userId,new Date());
          int status=chatService.addNewChat(chat);
          if (status!=0)return RestBean.success("cg");
          else return RestBean.failure(503,"error");
      }
  
  
      @GetMapping("/getChatByUserIdAndToUserId")
      public RestBean<List<Chat>>getChatByUserIdAndToUserId(HttpServletRequest request,
                                                            @RequestParam("toUserId") Integer toUserId)
      {
          Integer userId=(Integer)request.getAttribute("id");
          List<Chat> chatList=chatService.getChatByUserIdAndToUserId(userId,toUserId);
          return RestBean.success("cg",chatList);
      }
  
      @PostMapping("/CreateGroup")
      public RestBean<String> CreateGroup(HttpServletRequest request,
                                      @RequestParam("groupName")String groupName)
      {
          Integer userId=(Integer)request.getAttribute("id");
          GroupInformation groupInformation=new GroupInformation(groupName,userId,new Date());
          int groupId=chatService.CreateGroup(groupInformation);
          GroupUser groupUser=new GroupUser(groupId,userId,new Date());
          int status=chatService.JoinGroup(groupUser);
          if (status!=0)return RestBean.success("cg");
          else return RestBean.failure(503,"error");
      }
  
      @PostMapping("/JoinGroup")
      public RestBean<String> JoinGroup(@RequestParam("userId") Integer userId,
                                        @RequestParam("groupId")Integer groupId)
      {
  //        Integer userId=(Integer)request.getAttribute("id");
          GroupUser groupUser=new GroupUser(groupId,userId,new Date());
          int status=chatService.JoinGroup(groupUser);
          if (status!=0)return RestBean.success("cg");
          else return RestBean.failure(503,"error");
      }
  
  
      @GetMapping("/getGroupByUserId")
      public RestBean<List<GroupUserAndInformation>> getGroupByUserId(HttpServletRequest request)
      {
          Integer userId=(Integer)request.getAttribute("id");
          List<GroupUserAndInformation> chatList=chatService.getGroupByUserId(userId);
          return RestBean.success("cg",chatList);
      }
  
      @PostMapping("/addNewGroupChat")
      public RestBean<String> addNewGroupChat(HttpServletRequest request,
                                              @RequestParam("groupId")Integer groupId,
                                              @RequestParam("message")String message)
      {
          Integer sender=(Integer)request.getAttribute("id");
          GroupMessage groupMessage=new GroupMessage(groupId,message,sender,new Date());
          int status=chatService.addNewGroupChat(groupMessage);
          if (status!=0)return RestBean.success("cg");
          else return RestBean.failure(503,"error");
      }
  
  
      @GetMapping("/getGroupByGroupId")
      public RestBean<List<GroupMessageAndUser>> getGroupByGroupId(@RequestParam("groupId")Integer groupId)
      {
          List<GroupMessageAndUser> groupMessageAndUsers=chatService.getGroupByGroupId(groupId);
          return RestBean.success("gm",groupMessageAndUsers);
      }
  
      @GetMapping("/getGroupUserByGroupId")
      public RestBean<List<GroupUser>> getGroupUserByGroupId(@RequestParam("groupId")Integer groupId)
      {
          List<GroupUser> groupUsers=chatService.getGroupUserByGroupId(groupId);
          return RestBean.success("gu",groupUsers);
      }
  }
  ```

### WebSocket配置 🌐

- `WsServerEndpoint` 是核心通信类

- 功能亮点：

  - 使用userId管理会话
  - 支持点对点私聊
  - 自动处理JSON消息
  - 用户离线自动清理会话

- **WebSocketConfig类**

  ```
  @Configuration
  public class WebSocketConfig {
      @Bean
      public ServerEndpointExporter serverEndpointExporter(){
          return new ServerEndpointExporter();
      }
  }
  ```

- **WsServerEndpoint类**

  ```
  //监听websocket地址（/myWs）
  @ServerEndpoint("/myWs")
  @Component
  @Slf4j
  public class WsServerEndpoint {
  
      static Map<String, Session> sessionMap = new ConcurrentHashMap<>();
  
      // 建立连接
      @OnOpen
      public void onOpen(Session session) {
          // 获取前端传递的 userId 参数
          String userId = session.getRequestParameterMap().get("userId").getFirst();
          sessionMap.put(userId, session); // 使用 userId 作为键
          log.info("websocket is open, userId: {}",userId);
      }
  
      // 收到客户端消息
      @OnMessage
      public void onMessage(String message, Session session) {
          log.info("收到WebSocket消息: {}", message);
          try {
              // 尝试解析 JSON 消息
              ObjectMapper mapper = new ObjectMapper();
              Map<String, Object> msgMap = mapper.readValue(message, Map.class);
              log.info("解析后的消息内容: {}", msgMap);
              
              String toUserId = String.valueOf(msgMap.get("toUserId")); // 将数字转换为字符串
              String content = (String) msgMap.get("content");
              String fromUserId = session.getRequestParameterMap().get("userId").getFirst();
              
              log.info("处理消息 - 发送者: {}, 接收者: {}, 内容: {}", fromUserId, toUserId, content);
  
              // 检查目标用户是否存在
              if (sessionMap.containsKey(toUserId)) {
                  Session targetSession = sessionMap.get(toUserId);
                  targetSession.getBasicRemote().sendText("用户 " + fromUserId + " 对你说: " + content);
                  log.info("消息已发送给用户: {}", toUserId);
              } else {
                  // 可选：通知发送者目标用户不在线
                  session.getBasicRemote().sendText("用户 " + toUserId + " 不在线");
                  log.info("目标用户不存在: {}", toUserId);
              }
          } catch (Exception e) {
              log.error("消息处理失败: {}", e.getMessage());
              e.printStackTrace(); // 打印完整的错误堆栈
              // 如果不是JSON格式，则广播消息给所有用户
              String fromUserId = session.getRequestParameterMap().get("userId").getFirst();
              String broadcastMessage = "用户 " + fromUserId + " 广播消息: " + message;
              broadcast(broadcastMessage);
              log.info("收到非JSON格式消息，已广播: {}", message);
          }
      }
  
      // 关闭连接
      @OnClose
      public void onClose(Session session) {
          // 获取前端传递的 userId 参数
          String userId = session.getRequestParameterMap().get("userId").getFirst();
          sessionMap.remove(userId); // 使用 userId 移除对应的 Session
          log.info("websocket is close, userId: {}", userId);
      }
  
      // 广播消息给所有客户端
      private void broadcast(String message) {
          sessionMap.forEach((id, session) -> {
              if (session.isOpen()) {
                  try {
                      session.getBasicRemote().sendText(message);
                  } catch (IOException e) {
                      log.error("广播消息失败: {}", e.getMessage());
                  }
              }
          });
      }
  
      //定时任务，每隔两秒像客户端发送一个心跳
  //    @Scheduled(fixedRate = 2000)
  //    public void sendMessage() throws IOException {
  //        for(String key:sessionMap.keySet()){
  //            sessionMap.get(key).getBasicRemote().sendText("心跳");
  //        }
  //    }
  }
  ```

## 前端部分 🎨

### WebSocket商店 🏪

- 使用Pinia管理WebSocket状态

- 核心功能：

  - 初始化连接(`initWebSocket`)
  - 发送消息(`sendMsg`)
  - 关闭连接(`closeWebSocket`)
  - 消息回调处理(`setMessageCallback`)

- **建立连接+断开连接**：

  ```
  const wsStore = useWebSocketStore();
  
  // 添加beforeunload事件处理
  const handleBeforeUnload = () => {
    wsStore.closeWebSocket();
  };
  
  // 监听用户信息变化
  watch(() => userStore.user, (newUser) => {
    if (newUser) {
      wsStore.initWebSocket(newUser.id);
      window.addEventListener('beforeunload', handleBeforeUnload);
    }
  }, { immediate: true });
  
  //-------------------------------------------------
  onUnmounted(()=>{
    // 移除beforeunload事件监听器
    window.removeEventListener('beforeunload', handleBeforeUnload);
    wsStore.closeWebSocket();
  })
  ```

- **WebSocket代码**

  ```
  import { defineStore } from 'pinia'
  import { ref } from 'vue'
  import { post } from "@/net/index.js"
  
  export const useWebSocketStore = defineStore('websocket', () => {
    const ws = ref(null)
    const messageCallback=ref(null);
  
    const UpdateOnline = (online) => {
      post("api/user/UpdateOnline", {
        isOnline: online,
      }, (message) => {
        console.log("上线状态更新")
      })
    }
  
    const initWebSocket = (userId) => {
      // 如果已经存在连接且连接状态正常，则不重新连接
      if (ws.value && ws.value.readyState === WebSocket.OPEN) {
        return;
      }
      console.log("开始连接WebSocket")
      ws.value = new WebSocket(`ws://localhost:8080/myWs?userId=${userId}`)
      ws.value.onopen = () => {
        UpdateOnline("true")
        console.log("WebSocket连接成功！")
      }
      // 在 WebSocket 连接初始化时添加消息接收处理
      ws.value.onmessage = (event) => {
        console.log("收到WebSocket消息:", event.data)
        //检查是否已经设置了回调函数。如果已经设置了回调函数，那么就执行这个回调函数，并将收到的消息数据传递给它。
        if (messageCallback.value) {
          messageCallback.value(event.data)//调用 messageCallback.value 函数，并将 receivedData（收到的消息数据）作为参数传递给它。
        }
      }
    }
  
    const sendMsg = (toUserId,question) => {
      const privateMsg = {
        toUserId: toUserId,
        content: question
      };
      console.log(toUserId+" "+question);
      ws.value.send(JSON.stringify(privateMsg));
    }
  
    const closeWebSocket = () => {
      if (ws.value) {
        UpdateOnline("false");
        ws.value.close();
        ws.value = null;
      }
    }
    // 添加设置回调函数的方法
  const setMessageCallback = (callback) => {
    messageCallback.value = callback//可以理解为messageCallback现在就是传入的回调函数
  }
  
  
    return {
      ws,
      initWebSocket,
      closeWebSocket,
      sendMsg,
      setMessageCallback
    }
  }) 
  ```

### 群聊界面 🖥️

- 双栏布局：聊天区+联系人列表

- 功能亮点：

  - 区分私聊和群聊模式
  - 消息自动滚动到底部
  - 精美的用户头像显示
  - 群组创建和成员邀请功能

  ```
  <script setup>
  import {nextTick, onMounted, reactive, ref, watch} from "vue";
  import {get, post} from "@/net/index.js";
  import { UserOutlined } from '@ant-design/icons-vue';
  import {userUserStore} from "@/stores/userStore.js";
  import {useWebSocketStore} from "@/stores/websocketStore.js";
  import {message, Modal} from "ant-design-vue";
  import { h } from 'vue';
  import {ElMessageBox} from "element-plus";
  
  
  const [messageApi, contextHolder] = message.useMessage();
  const userStore=userUserStore();
  const wsStore = useWebSocketStore();
  const ChatMessage=ref([]);
  const SelectContent=ref('');
  const user=reactive({
    user:[],
    question:'',
    toUserId:0,
    avatar1:'',
    avatar2:'',
  })
  const getAllUsers=()=>{
    get('api/user/getAllUsers',{},(message,data)=>{
      user.user=data;
    })
  }
  
  const getChatByUserIdAndToUserId=()=>{
    get('/api/chat/getChatByUserIdAndToUserId',{
      toUserId:user.toUserId,
    },(message,data)=>{
      const Message=Array.isArray(data)?data:[data];
      Message.forEach((content)=>{
        ChatMessage.value.push({userId:content.sender,content:content.message})
      })
      nextTick(() => scrollToBottom());
    })
  }
  
  const selectUser=(userId,avatar)=>{
    ChatMessage.value=[];
    user.toUserId=userId;
    SelectContent.value='好友';
    user.avatar1=userStore.user.avator;
    user.avatar2=avatar
    getChatByUserIdAndToUserId();
  }
  
  const addNewPrivateChat=()=>{
    post('/api/chat/PrivateChat',{
      toUserId:user.toUserId,
      message:user.question,
    },(message)=>{
      messageApi.success("发送成功");
      console.log('发送的消息:', {userId:userStore.user.id,content:user.question});
      ChatMessage.value.push({userId:userStore.user.id,content:user.question});
      wsStore.sendMsg(user.toUserId,user.question);
      user.question='';
    })
  }
  
  const getAvatarByUserId=(userId)=>{
    //Promise 用于处理异步操作，resolve 和 reject 是回调函数，分别用于处理成功和失败的情况。
    return new Promise((resolve, reject) => {
      get('/api/user/getAvatarByUserId',{
        userId:userId,
      },(message,data)=>{
        resolve(data);
      })
    });
  }
  
  // 添加消息处理函数
  const handleWebSocketMessage = async (data) => {
    console.log("收到WebSocket消息:", data);
    // 解析消息格式 "用户 xxx 对你说: xxx"
    const match = data.match(/用户 (\d+) 对你说: (.+)/);
    if (match) {
      const [, fromUserId, content] = match;
      try {
  
        if(SelectContent.value==='好友'){
          console.log("当前是好友聊天模式");
          ChatMessage.value.push({
            userId: parseInt(fromUserId),
            content: content,
          });
        }
        else {
          console.log("当前是群聊模式");
          ChatMessage.value.push({
            userId: parseInt(fromUserId),
            content: content,
            avatar:await getAvatarByUserId(parseInt(fromUserId))
          });
        }
      } catch (error) {
        console.error("获取头像失败:", error);
      }
    }
  }
  
  //引用需要滚动的内容
  const messageContainer = ref(null);
  // 添加一个滚动到底部的方法
  const scrollToBottom = () => {
    if (messageContainer.value) {
      messageContainer.value.scrollTop = messageContainer.value.scrollHeight;
    }
  };
  // 在消息更新后调用滚动方法
  watch(ChatMessage, () => {
    nextTick(() => {
      scrollToBottom();
    });
  });
  
  watch(()=>wsStore.ws,(newWs,oldValue)=>{
    if(newWs!==oldValue)
    getAllUsers();
  })
  
  //-------------------------------------------群聊---------------------------------
  const modal2Visible = ref(false);
  const Groups=reactive({
    name:'',
    group:[],
    toGroupId:0,
    IsInvite:false,
    selectedUsers:[],//被邀请的用户
  })
  //群头像
  const groupImage=[
      'https://tse3-mm.cn.bing.net/th/id/OIP-C.cQYUayK9NKywYZ05d8Pp9QAAAA?rs=1&pid=ImgDetMain',
      'https://pic4.zhimg.com/50/v2-171afda2c73e30ac862eff613c5eedbc_hd.jpg?source=1940ef5c',
      'https://pic4.zhimg.com/v2-8699e5372b3d71cfc90027015c4a7a09_r.jpg?source=1940ef5c',
      'https://tse1-mm.cn.bing.net/th/id/OIP-C.GAC8EhRgjiF3End5bTOspwHaHa?rs=1&pid=ImgDetMain'
  ]
  
  //获取群里面的成员
  const getGroupUserByGroupId=()=>{
    Groups.selectedUsers=[];
    get('api/chat/getGroupUserByGroupId',{
      groupId:Groups.toGroupId,
    },(message,data)=>{
      const GroupUser=Array?data:[data];
      GroupUser.forEach((user)=>{
        Groups.selectedUsers.push({userId:user.userId,IsEdit:false});
      })
    })
  }
  
  //选择需要邀请的人
  const SelectUser=(userId)=> {
    // 查找是否已存在该用户
    const userIndex = Groups.selectedUsers.findIndex(u => u.userId === userId);
    if (userIndex !== -1) {
      // 如果用户已存在
      if (Groups.selectedUsers[userIndex].IsEdit) {
        // 只有可编辑的用户才能被移除
        Groups.selectedUsers.splice(userIndex, 1);
      }
      // 不可编辑的用户不做任何操作（无法取消选择）
    } else {
      // 如果用户不存在，则添加（带可编辑状态）
      Groups.selectedUsers.push({
        userId: userId,
        IsEdit: true  // 传入是否可编辑的标志
      });
    }
  }
  //打开创建群聊的对话框
  const OpenModalAndGetGroupName=(id)=>{
    modal2Visible.value = true ;
    Groups.toGroupId=id;
    getGroupUserByGroupId();
  }
  
  //将邀请的人加入群聊
  const SelectUserGoGroup=()=>{
    modal2Visible.value=false;
    Groups.selectedUsers.forEach((SelectUser)=>{
      if(SelectUser.IsEdit)
      {
        post('api/chat/JoinGroup',{
          userId:parseInt(SelectUser.userId),
          groupId: Groups.toGroupId,
        },(message)=>{
          messageApi.success("加入群聊成功！");
          Groups.selectedUsers=[];
          getGroupUserByGroupId();
        })
      }
    })
  }
  
  //获取加入的群聊
  const getGroupByUserId=()=>{
    get("/api/chat/getGroupByUserId",{},(message,data)=>{
      Groups.group=data;
    })
  }
  //选择群聊
  const SelectGroup=(groupId)=>{
    ChatMessage.value=[];
    Groups.toGroupId=groupId;
    SelectContent.value='群聊';
    getGroupByGroupId();
  }
  //创建群聊
  const GoGroup=()=>{
    post('api/chat/CreateGroup',{
      groupName:Groups.name,
    },(message)=>{
      messageApi.success("创建群聊成功！");
      Groups.group.push({groupName:Groups.name})
    })
  }
  //获取群聊信息
  const getGroupByGroupId=()=>{
    get("/api/chat/getGroupByGroupId",{
      groupId:Groups.toGroupId
    },(message,data)=>{
      const GroupMessage=Array?data:[data];
      GroupMessage.forEach((group)=>{
        ChatMessage.value.push({userId:group.groupMessage.sender,content:group.groupMessage.message,avatar:group.avatar});
      })
    })
  }
  //发送群聊信息
  const addNewGroupChat=()=>{
    getGroupUserByGroupId();
    post('/api/chat/addNewGroupChat',{
      groupId:Groups.toGroupId,
      message:user.question,
    },(message)=>{
      messageApi.success("发送成功！");
      ChatMessage.value.push({userId:userStore.user.id,content:user.question,avatar:userStore.user.avator});
  
      console.log(Groups.selectedUsers)
      Groups.selectedUsers.forEach((SelectUser)=>{
        console.log("执行了吗")
        if(SelectUser.userId!==userStore.user.id)
        {
          wsStore.sendMsg(SelectUser.userId,user.question);
        }
      })
      user.question='';
    })
  }
  const colors = [
    'pink',
    'red',
    'yellow',
    'orange',
    'cyan',
    'green',
    'blue',
    'purple',
    'geekblue',
    'magenta',
    'volcano',
    'gold',
    'lime',
  ];
  //创建群聊
  const GroupInformation=()=>{
    ElMessageBox.prompt('请输入群聊名称', '提示', {
      confirmButtonText: '确定',
      cancelButtonText: '取消',
    })
        .then(({ value }) => {
          Groups.name=value;
          GoGroup();
        })
        .catch(() => {
  
        });
  }
  //区分发送类型（好友/群聊）
  const SendMessage=()=>{
    if(SelectContent.value==="好友")
    {
      addNewPrivateChat();
    }
    else if(SelectContent.value==="群聊")
    {
      addNewGroupChat();
    }
  }
  
  onMounted(()=>{
    ChatMessage.value.push({userId:-1,content:"欢迎来到fz的聊天界面！"})
    getAllUsers();
    getGroupByUserId();
    getChatByUserIdAndToUserId()
    wsStore.setMessageCallback(handleWebSocketMessage);
  })
  
  </script>
  
  <template>
    <contextHolder/>
  <div class="grid grid-cols-[7fr,2fr] w-full max-h-screen p-4 gap-6">
  <!--  对话栏-->
    <div class="grid grid-rows-[11fr,2fr] h-full bg-opacity-100 gap-6">
  <!--    消息展示-->
      <div ref="messageContainer" class="w-full h-full  overflow-y-auto scrollbar-hide bg-white rounded-xl shadow-gray-500 shadow-md dark:bg-opacity-5 " style="max-height: 75vh">
        <div v-if="SelectContent==='好友'" v-for="message in ChatMessage" class="gap-4 ">
          <div v-if="message.userId===userStore.user.id" class="w-full flex justify-end p-2 gap-2">
            <p class="max-w-4/5 flex flex-wrap my-auto border-2 border-gray-300 rounded-xl p-3">{{message.content}}</p>
            <a-avatar  :src="user.avatar1"  size="large"/>
          </div>
          <div v-else class="w-full flex  justify-start p-2 gap-2">
            <a-avatar  :src="user.avatar2===''?'https://www.keaitupian.cn/cjpic/frombd/0/253/3730063558/1625161022.jpg':user.avatar2"  size="large"/>
            <p class="max-w-4/5 flex flex-wrap my-auto border-2 border-gray-300 rounded-xl p-3 ">{{message.content}}</p>
          </div>
        </div>
        <div v-else v-for="message in ChatMessage" class="gap-4 ">
          <div v-if="message.userId===userStore.user.id" class="w-full flex justify-end p-2 gap-2">
            <p class="max-w-4/5 flex flex-wrap my-auto border-2 border-gray-300 rounded-xl p-3">{{message.content}}</p>
            <a-avatar  :src="message.avatar"  size="large"/>
          </div>
          <div v-else class="w-full flex  justify-start p-2 gap-2">
            <a-avatar  :src="message.avatar==null?'https://www.keaitupian.cn/cjpic/frombd/0/253/3730063558/1625161022.jpg':message.avatar"  size="large"/>
            <p class="max-w-4/5 flex flex-wrap my-auto border-2 border-gray-300 rounded-xl p-3 ">{{message.content}}</p>
          </div>
        </div>
      </div>
  <!--    输入框-->
      <a-textarea v-if="user.toUserId!==0||Groups.toGroupId!==0" @keyup.enter="SendMessage" v-model:value="user.question" class="shadow-gray-500 shadow-md bg-white dark:bg-opacity-5" :rows="4"  placeholder="maxlength is 1000" :maxlength="10000" />
      <a-alert v-else
          message="Informational Notes"
          description="Please select the user you want to chat with first."
          type="info"
          show-icon
      />
    </div>
  <!--  侧边栏-->
    <div class="w-full min-h-full flex flex-wrap bg-white rounded-xl shadow-gray-500 shadow-md dark:bg-opacity-5">
      <!--    功能栏-->
      <div class="w-full p-2 border-b border-gray-200 flex flex-nowrap gap-2">
        <a-tooltip title="群聊" :color="colors[12]">
          <a-button @click="GroupInformation">
            <svg class="size-6 hover:scale-105 active:scale-90" xmlns="http://www.w3.org/2000/svg" width="128" height="128" viewBox="0 0 24 24">
              <g fill="none" stroke="currentColor" stroke-width="2"><path stroke-linecap="round" stroke-linejoin="round" d="m16.719 19.752l-.64-5.124A3 3 0 0 0 13.101 12h-2.204a3 3 0 0 0-2.976 2.628l-.641 5.124A2 2 0 0 0 9.266 22h5.468a2 2 0 0 0 1.985-2.248"/><circle cx="12" cy="5" r="3"/><circle cx="4" cy="9" r="2"/><circle cx="20" cy="9" r="2"/><path stroke-linecap="round" stroke-linejoin="round" d="M4 14h-.306a2 2 0 0 0-1.973 1.671l-.333 2A2 2 0 0 0 3.361 20H7m13-6h.306a2 2 0 0 1 1.973 1.671l.333 2A2 2 0 0 1 20.639 20H17"/></g>
            </svg>
          </a-button>
        </a-tooltip>
  
  <!--      搜索-->
        <svg class="size-8 hover:scale-105 active:scale-90" xmlns="http://www.w3.org/2000/svg" width="128" height="128" viewBox="0 0 24 24">
          <g fill="none" stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" color="currentColor"><path d="M22 12c0-5.523-4.477-10-10-10S2 6.477 2 12s4.477 10 10 10s10-4.477 10-10"/><path d="M14.4 14.4L16 16m-.8-4.4a3.6 3.6 0 1 0-7.2 0a3.6 3.6 0 0 0 7.2 0"/></g>
        </svg>
  <!--      铃铛-->
        <svg class="size-8 hover:scale-105 active:scale-90" xmlns="http://www.w3.org/2000/svg" width="128" height="128" viewBox="0 0 24 24">
          <path fill="none" stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M13.5 17H4a4 4 0 0 0 2-3v-3a7 7 0 0 1 4-6a2 2 0 1 1 4 0a7 7 0 0 1 4 6v1m-9 5v1a3 3 0 0 0 4.368 2.67M19 16l-2 3h4l-2 3"/>
        </svg>
      </div>
      <!--    好友列表容器 -->
      <div class="w-full h-[calc(100%-3rem)] overflow-y-auto">
        <!--    好友-->
        <div v-for="user in user.user" class="w-full h-fit justify-start flex flex-nowrap p-4 gap-2">
          <a-avatar  :src="user.avator"  size="large" />
          <p class="flex flex-nowrap items-center justify-center my-auto">{{user.username}}</p>
          <svg v-if="user.isOnline==='true'" class="size-6 flex text-green-600 justify-end my-auto " xmlns="http://www.w3.org/2000/svg" width="128" height="128" viewBox="0 0 16 16">
            <path fill="currentColor" d="M8 14a1.5 1.5 0 1 1 0-3a1.5 1.5 0 0 1 0 3m0-1a.5.5 0 1 0 0-1a.5.5 0 0 0 0 1m3.189-3.64a.5.5 0 0 1-.721.692A3.4 3.4 0 0 0 8 9c-.937 0-1.813.378-2.453 1.037a.5.5 0 0 1-.717-.697A4.4 4.4 0 0 1 8 8c1.22 0 2.361.497 3.189 1.36m2.02-2.14a.5.5 0 1 1-.721.693A6.2 6.2 0 0 0 8 6a6.2 6.2 0 0 0-4.46 1.885a.5.5 0 0 1-.718-.697A7.2 7.2 0 0 1 8 5a7.2 7.2 0 0 1 5.21 2.22m2.02-2.138a.5.5 0 0 1-.721.692A9 9 0 0 0 8 3a9 9 0 0 0-6.469 2.734a.5.5 0 1 1-.717-.697A10 10 0 0 1 8 2a10 10 0 0 1 7.23 3.082"/>
          </svg>
          <svg v-else class="size-6 flex justify-end my-auto" xmlns="http://www.w3.org/2000/svg" width="128" height="128" viewBox="0 0 16 16">
            <path fill="currentColor" d="m6.517 12.271l1.254-1.254Q7.883 11 8 11a1.5 1.5 0 1 1-1.483 1.271m2.945-2.944l.74-.74c.361.208.694.467.987.772a.5.5 0 0 1-.721.693a3.4 3.4 0 0 0-1.006-.725m2.162-2.163l.716-.715q.463.349.87.772a.5.5 0 1 1-.722.692a6.3 6.3 0 0 0-.864-.749M7.061 6.07A6.2 6.2 0 0 0 3.54 7.885a.5.5 0 0 1-.717-.697a7.2 7.2 0 0 1 5.309-2.187zm6.672-1.014l.71-.71q.411.346.786.736a.5.5 0 0 1-.721.692a9 9 0 0 0-.775-.718m-3.807-1.85A9 9 0 0 0 8 3a9 9 0 0 0-6.469 2.734a.5.5 0 1 1-.717-.697A10 10 0 0 1 8 2c.944 0 1.868.131 2.75.382zM8 13a.5.5 0 1 0 0-1a.5.5 0 0 0 0 1m-5.424 1a.5.5 0 0 1-.707-.707L14.146 1.146a.5.5 0 0 1 .708.708z"/>
          </svg>
          <svg @click="selectUser(user.id,user.avator)" class="ml-auto size-6 flex items-center justify-end mr-0 my-auto hover:text-blue-600 hover:scale-110 active:scale-90" xmlns="http://www.w3.org/2000/svg"  viewBox="0 0 24 24">
            <g fill="none"><path fill="currentColor" d="m13.087 21.388l.645.382zm.542-.916l-.646-.382zm-3.258 0l-.645.382zm.542.916l.646-.382zm-8.532-5.475l.693-.287zm5.409 3.078l-.013.75zm-2.703-.372l-.287.693zm16.532-2.706l.693.287zm-5.409 3.078l-.012-.75zm2.703-.372l.287.693zm.7-15.882l-.392.64zm1.65 1.65l.64-.391zM4.388 2.738l-.392-.64zm-1.651 1.65l-.64-.391zM9.403 19.21l.377-.649zm4.33 2.56l.541-.916l-1.29-.764l-.543.916zm-4.007-.916l.542.916l1.29-.764l-.541-.916zm2.715.152a.52.52 0 0 1-.882 0l-1.291.764c.773 1.307 2.69 1.307 3.464 0zM10.5 2.75h3v-1.5h-3zm10.75 7.75v1h1.5v-1zm-18.5 1v-1h-1.5v1zm-1.5 0c0 1.155 0 2.058.05 2.787c.05.735.153 1.347.388 1.913l1.386-.574c-.147-.352-.233-.782-.278-1.441c-.046-.666-.046-1.51-.046-2.685zm6.553 6.742c-1.256-.022-1.914-.102-2.43-.316L4.8 19.313c.805.334 1.721.408 2.977.43zM1.688 16.2A5.75 5.75 0 0 0 4.8 19.312l.574-1.386a4.25 4.25 0 0 1-2.3-2.3zm19.562-4.7c0 1.175 0 2.019-.046 2.685c-.045.659-.131 1.089-.277 1.441l1.385.574c.235-.566.338-1.178.389-1.913c.05-.729.049-1.632.049-2.787zm-5.027 8.241c1.256-.021 2.172-.095 2.977-.429l-.574-1.386c-.515.214-1.173.294-2.428.316zm4.704-4.115a4.25 4.25 0 0 1-2.3 2.3l.573 1.386a5.75 5.75 0 0 0 3.112-3.112zM13.5 2.75c1.651 0 2.837 0 3.762.089c.914.087 1.495.253 1.959.537l.783-1.279c-.739-.452-1.577-.654-2.6-.752c-1.012-.096-2.282-.095-3.904-.095zm9.25 7.75c0-1.622 0-2.891-.096-3.904c-.097-1.023-.299-1.862-.751-2.6l-1.28.783c.285.464.451 1.045.538 1.96c.088.924.089 2.11.089 3.761zm-3.53-7.124a4.25 4.25 0 0 1 1.404 1.403l1.279-.783a5.75 5.75 0 0 0-1.899-1.899zM10.5 1.25c-1.622 0-2.891 0-3.904.095c-1.023.098-1.862.3-2.6.752l.783 1.28c.464-.285 1.045-.451 1.96-.538c.924-.088 2.11-.089 3.761-.089zM2.75 10.5c0-1.651 0-2.837.089-3.762c.087-.914.253-1.495.537-1.959l-1.279-.783c-.452.738-.654 1.577-.752 2.6C1.25 7.61 1.25 8.878 1.25 10.5zm1.246-8.403a5.75 5.75 0 0 0-1.899 1.899l1.28.783a4.25 4.25 0 0 1 1.402-1.403zm7.02 17.993c-.202-.343-.38-.646-.554-.884a2.2 2.2 0 0 0-.682-.645l-.754 1.297c.047.028.112.078.224.232c.121.166.258.396.476.764zm-3.24-.349c.44.008.718.014.93.037c.198.022.275.054.32.08l.754-1.297a2.2 2.2 0 0 0-.909-.274c-.298-.033-.657-.038-1.069-.045zm6.498 1.113c.218-.367.355-.598.476-.764c.112-.154.177-.204.224-.232l-.754-1.297c-.29.17-.5.395-.682.645c-.173.238-.352.54-.555.884zm1.924-2.612c-.412.007-.771.012-1.069.045c-.311.035-.616.104-.909.274l.754 1.297c.045-.026.122-.058.32-.08c.212-.023.49-.03.93-.037z"/><path stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 11h.009m3.982 0H12m3.991 0H16"/></g>
          </svg>
        </div>
        <!--    好友-->
  <!--      群聊-->
        <div v-for="(group,index) in Groups.group" :key="index" class="w-full h-fit justify-start flex flex-nowrap p-4 gap-2">
          <a-avatar  :src="groupImage[index]"  size="large" />
          <p class="flex flex-nowrap items-center justify-center my-auto">{{group.groupName}}</p>
          <svg @click="OpenModalAndGetGroupName(group.groupId)" class=" size-6 flex items-center justify-end mr-0 my-auto hover:text-blue-600 hover:scale-110 active:scale-90" xmlns="http://www.w3.org/2000/svg" width="128" height="128" viewBox="0 0 48 48">
            <g fill="none" stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="4"><circle cx="24" cy="12" r="8"/><path d="M42 44c0-9.941-8.059-18-18-18S6 34.059 6 44m13-5h10m-5-5v10"/></g>
          </svg>
          <a-modal
              v-model:open="modal2Visible"
              title="邀请好友加入群聊！"
              centered
              @ok="SelectUserGoGroup"
          >
            <div class="scrollbar-hide overflow-y-auto"  style="max-height: 70vh">
              <div  v-for="user in user.user" class="w-full  justify-start flex flex-nowrap p-4 gap-4" >
                <a-avatar  :src="user.avator"  size="large" class="flex flex-nowrap my-auto select-none"/>
                <p class="flex flex-nowrap items-center justify-center my-auto select-none">{{user.username}}</p>
                <svg @click="SelectUser(user.id)" v-if="Groups.selectedUsers.findIndex(u => u.userId === user.id)===-1" class="ml-auto size-6 flex items-center justify-end mr-0 my-auto hover:text-blue-600 hover:scale-110 active:scale-90" xmlns="http://www.w3.org/2000/svg" width="128" height="128" viewBox="0 0 24 24">
                  <path fill="currentColor" d="M11 13H5v-2h6V5h2v6h6v2h-6v6h-2z"/>
                </svg>
                <svg @click="SelectUser(user.id)" v-else class="ml-auto text-green-600 size-6 flex items-center justify-end mr-0 my-auto hover:text-blue-600 hover:scale-110 active:scale-90" xmlns="http://www.w3.org/2000/svg" width="128" height="128" viewBox="0 0 48 48">
                  <g fill="none" stroke="currentColor" stroke-linecap="round" stroke-width="4"><path d="M33 7.263A18.9 18.9 0 0 0 24 5C13.507 5 5 13.507 5 24s8.507 19 19 19a18.9 18.9 0 0 0 8-1.761"/><path stroke-linejoin="round" d="M31 30h12m-28-8l7 7l19-18m-4 13v12"/></g>
                </svg>
              </div>
            </div>
          </a-modal>
          <svg @click="SelectGroup(group.groupId)" class="ml-auto size-6 flex items-center justify-end mr-0 my-auto hover:text-blue-600 hover:scale-110 active:scale-90" xmlns="http://www.w3.org/2000/svg"  viewBox="0 0 24 24">
            <g fill="none"><path fill="currentColor" d="m13.087 21.388l.645.382zm.542-.916l-.646-.382zm-3.258 0l-.645.382zm.542.916l.646-.382zm-8.532-5.475l.693-.287zm5.409 3.078l-.013.75zm-2.703-.372l-.287.693zm16.532-2.706l.693.287zm-5.409 3.078l-.012-.75zm2.703-.372l.287.693zm.7-15.882l-.392.64zm1.65 1.65l.64-.391zM4.388 2.738l-.392-.64zm-1.651 1.65l-.64-.391zM9.403 19.21l.377-.649zm4.33 2.56l.541-.916l-1.29-.764l-.543.916zm-4.007-.916l.542.916l1.29-.764l-.541-.916zm2.715.152a.52.52 0 0 1-.882 0l-1.291.764c.773 1.307 2.69 1.307 3.464 0zM10.5 2.75h3v-1.5h-3zm10.75 7.75v1h1.5v-1zm-18.5 1v-1h-1.5v1zm-1.5 0c0 1.155 0 2.058.05 2.787c.05.735.153 1.347.388 1.913l1.386-.574c-.147-.352-.233-.782-.278-1.441c-.046-.666-.046-1.51-.046-2.685zm6.553 6.742c-1.256-.022-1.914-.102-2.43-.316L4.8 19.313c.805.334 1.721.408 2.977.43zM1.688 16.2A5.75 5.75 0 0 0 4.8 19.312l.574-1.386a4.25 4.25 0 0 1-2.3-2.3zm19.562-4.7c0 1.175 0 2.019-.046 2.685c-.045.659-.131 1.089-.277 1.441l1.385.574c.235-.566.338-1.178.389-1.913c.05-.729.049-1.632.049-2.787zm-5.027 8.241c1.256-.021 2.172-.095 2.977-.429l-.574-1.386c-.515.214-1.173.294-2.428.316zm4.704-4.115a4.25 4.25 0 0 1-2.3 2.3l.573 1.386a5.75 5.75 0 0 0 3.112-3.112zM13.5 2.75c1.651 0 2.837 0 3.762.089c.914.087 1.495.253 1.959.537l.783-1.279c-.739-.452-1.577-.654-2.6-.752c-1.012-.096-2.282-.095-3.904-.095zm9.25 7.75c0-1.622 0-2.891-.096-3.904c-.097-1.023-.299-1.862-.751-2.6l-1.28.783c.285.464.451 1.045.538 1.96c.088.924.089 2.11.089 3.761zm-3.53-7.124a4.25 4.25 0 0 1 1.404 1.403l1.279-.783a5.75 5.75 0 0 0-1.899-1.899zM10.5 1.25c-1.622 0-2.891 0-3.904.095c-1.023.098-1.862.3-2.6.752l.783 1.28c.464-.285 1.045-.451 1.96-.538c.924-.088 2.11-.089 3.761-.089zM2.75 10.5c0-1.651 0-2.837.089-3.762c.087-.914.253-1.495.537-1.959l-1.279-.783c-.452.738-.654 1.577-.752 2.6C1.25 7.61 1.25 8.878 1.25 10.5zm1.246-8.403a5.75 5.75 0 0 0-1.899 1.899l1.28.783a4.25 4.25 0 0 1 1.402-1.403zm7.02 17.993c-.202-.343-.38-.646-.554-.884a2.2 2.2 0 0 0-.682-.645l-.754 1.297c.047.028.112.078.224.232c.121.166.258.396.476.764zm-3.24-.349c.44.008.718.014.93.037c.198.022.275.054.32.08l.754-1.297a2.2 2.2 0 0 0-.909-.274c-.298-.033-.657-.038-1.069-.045zm6.498 1.113c.218-.367.355-.598.476-.764c.112-.154.177-.204.224-.232l-.754-1.297c-.29.17-.5.395-.682.645c-.173.238-.352.54-.555.884zm1.924-2.612c-.412.007-.771.012-1.069.045c-.311.035-.616.104-.909.274l.754 1.297c.045-.026.122-.058.32-.08c.212-.023.49-.03.93-.037z"/><path stroke="currentColor" stroke-linecap="round" stroke-linejoin="round" stroke-width="2" d="M8 11h.009m3.982 0H12m3.991 0H16"/></g>
          </svg>
        </div>
  <!--      群聊-->
      </div>
    </div>
  </div>
  </template>
  
  <style>
  /* 隐藏所有浏览器滚动条 */
  .scrollbar-hide::-webkit-scrollbar {
    display: none; /* Chrome/Safari/Edge */
  }
  
  .scrollbar-hide {
    -ms-overflow-style: none;  /* IE/Edge */
    scrollbar-width: none;     /* Firefox */
  }
  </style>
  ```

## 趣味知识点 🎯

1. **会话管理小聪明** 🧠
   - WebSocket用userId作为键存储会话，比随机sessionId更直观
   - 就像给每个用户发了个专属对讲机📻
2. **消息处理双模式** 🔄
   - 自动识别JSON格式消息，不是JSON就当广播处理
   - 就像邮差叔叔，能认出挂号信和普通信件的区别📨
3. **懒人滚动条** 🎢
   - 消息自动滚动到底部，不用手动拖动
   - 像坐过山车一样自动滑到最刺激的部分！
4. **群聊邀请小机关** 🎁
   - 点击"+"号弹出好友列表，像打开礼物盒一样挑选好友加入
5. **心跳检测** 💓
   - 注释掉的定时任务展示了如何实现心跳检测
   - 就像定期喊"你还活着吗？"，确保连接健康

## 系统趣事 🎭

1. **头像故事** 👤
   - 默认头像是个神秘黑衣人，像特工电影里的未露面角色
2. **错误处理** 🤦
   - 当用户不在线时，会收到"用户xxx不在线"提示
   - 就像打电话听到"您拨打的用户暂时无法接通"
3. **彩蛋功能** 🥚
   - 搜索和通知按钮已经预留，等待开发
   - 像游戏里的未解锁成就🔒

这个系统就像个数字化的咖啡馆，私聊是角落的两人小桌，群聊是中间的圆桌会议，WebSocket是穿梭其间的服务员，而你就是这家咖啡馆的老板！☕
