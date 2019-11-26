# 基本结构（暂定）
- common.proto ： 公共部分

- apiHttp.proto ：短连接协议

- apiTcp.proto ：长连接协议

# 公共协议部分

- 错误码
- 包名: **CommonProto.xxx**

```protobuf
enum ResultCode {
    DEFAULT = 0;                //保留字段，不使用
    OK = 1;                     //成功

   	ACCOUNT_NOT_EXIST = 2;      //账号不存在
   	ACCOUNT_EXIST = 3;          //账号已存在
   	ACCOUNT_VERIFY_FAIL = 4;    //账号 验证错误
   	TOKEN_FAIL = 5;             //token 验证错误

	FAIL = 9999;               //服务器 未定义的错误，客户端一般显示：服务器维护中，稍后再试
}
```



# Http 短连接协议



## 连接参数

- 访问地址：**http://192.168.1.106:7770/gamble**
- 数据请求格式： **POST**
- 参数 : **protobuff 二进制**
- 包名: **HttpProto.xxx**



## 收发协议

- **请求类型枚举**

  ```protobuf
  enum RequestType {
      REGISTER = 0;   //注册
      LOGIN = 1;      //登录
  }
  ```
  
- **C2S，统一的请求协议**

  ```protobuf
  message C2S_Cmd {
      RequestType type;
      bytes data;			//客户端 请求的信息内容
  }
  ```

- **S2C，统一的接收协议**

  ```protobuf
  message S2C_Cmd {
      ResultCode result;
      bytes data;     	    //客户端 接收的信息内容，可选字段，可以为空
  }
  ```



## 协议内容

- **账号类型**

  ```protobuf
  enum AccountType {
  	DEFAULT = 0;
  	ACCOUNT = 1;
  	GUEST = 2;
  	PHONE = 3;
  	MAIL = 4;
  	FACEBOOK = 5;
  }
  ```

### **登录**

  + C2S

    ```protobuf
    message C2S_Login {
    	AccountType type;
        string account;
        string password;
    }
    ```

  + S2C

    ```protobuf
    //	Successful Response
    message S2C_Login {
        string token;
    }
    
    //	Fail Response
    message S2C_Login {
    }
    ```

###  **注册**

  + C2S

    ```protobuf
    message C2S_Register {
    	AccountType type;
    	string account;
    	string password;
    }
    ```

  + S2C

    ```protobuf
    //	Successful Response
    //	Fail Response
    message S2C_Register {
    }
    ```
    

###  请求MQTT服务器列表 

  - C2S

    ```protobuf
    message C2S_MqttList {
    	string token;
    }
    ```

    

  - S2C

    ```protobuf
    message S2C_MqttList {
    }
    ```

    

## 范例

```lua
//	HttpProto 是 proto 的包名
//	CommonProto 是 proto 的包名

//	发送消息
local data = pb.encode
(
    "HttpProto.C2S_Login", 				
    {
        type = pb.enum("HttpProto.AccountType", "ACCOUNT"),
        account = "Test_Account",
        password = "Test_Password"
    }
)

local request = pb.encode
(
    "HttpProto.C2S_Cmd",
    {
        type = pb.enum("HttpProto.RequestType", "LOGIN"),
        data = data
    }
)

//	接收消息
local Response = pb.decode(
    "HttpProto.S2C_Cmd",
    data						// typeof(Response) == table
)	
if Response.result == pb.enum("CommonProto.ResultCode", "OK") then
    //	成功逻辑
    pb.decode(
        "HttpProto.S2C_Login", 
        Response.data			// typeof(Response.data) == table 
    )
else
    //	失败逻辑 
end
```



# 长连接协议  



## 连接参数

- 访问地址：**http://192.168.1.106:7771** (暂定)

- 包名: **TcpProto.xxx**

- 参数 : **protobuff 二进制**

- 长链接连接

  ```lua
  --	必须参数
  local string token = pb.enum("HttpProto.S2C_Login", "token")
  local string clientid = CS.SystemInfo.deviceUniqueIdentifier	//	已集成
  --	函数
  Network:KeepAliveConnect(token)
  --	回执: S2C_Cmd
  --	频道: VerifyLogin
  ```
  
  

## 收发协议

- **C2S，统一的请求协议**

  ```protobuf
  message C2S_Cmd {
      bytes data;				//请求的信息内容
  }
  ``````

- **S2C，统一的接收协议**

  ```protobuf
  message S2C_Cmd {
        ResultCode code;
        bytes data;			//客户端 接收的信息内容，可选字段，可以为空
  }
  ```

- **<u>协议内容为空即默认为没有data</u>**

- <u>**Push的协议, 默认没有code**</u>



## 协议内容

### 获取游戏列表信息

- Type: Requests and Responses

- Topic: **<u>GetGameWeight</u>**

- C2S

  ```protobuf
  message C2S_GetGameWeight {
  }
  ```
- S2C  

  ```protobuf
  message GameWeight {      
  	sint32 index;                  //  游戏ID      
  	sint32 type;                   //  游戏状态 
      sint32 weights;                //  游戏权重  
  }  
  
  //    Successful Response  
  message S2C_GetGameWeight {    
  	repeated GameWeight weight_list;  
  }
  
  //    Fail Response  
  message S2C_GetGameWeight {
  }
  ```

### 获取单个玩家的信息

- Type: Requests and Responses
+ Topic: **<u>GetUserInfo</u>**

+ C2S

  ```protobuf
  message C2S_GetUserInfo {
  	string user_id;		//	获取自己信息发送0, 获取别的玩家信息发送目标ID
  }
  ```

+ S2C

  ```protobuf
  //	Successful Response
  message S2C_GetUserInfo {
  	string user_id;		//	玩家ID
  	string portrait;	//	玩家头像 (URL)
  	string user_name;	//	玩家名字
  	sint32 vip;         //	VIP等级 vip = 0 无
  	sint64 vip_exp;		//	当前VIP经验值
  	sint32 level;		//	当前等级
  	sint64 level_exp;	//	当前等级经验值
  	sint64 chip;		//	筹码余额 
  	sint64 integral		//	积分余额
  	sint32 game_count;	//	已玩的游戏次数
  	sint64 start_time;	//	第一次进入游戏的时间
  }
  
  //	Fail Response
  message S2C_GetUserInfo {		
  }
  ```
  



### 选择游戏

- Type: Requests and Responses

- Topic: <u>**SelectGame**</u>
- C2S
    ```protobuf
     message C2S_SelectGame {
        sint32 game_id;		//	游戏ID, ID对应游戏通过读表同步
     }
    ```
- S2
  ```protobuf
  //	Successful Response
  //	Fail Response
  message S2C_SelectGame {
  }
  ```



- 

### 获取牌桌信息(暂定可跨游戏获取信息)

- Type: Requests and Responses
- Topic: <u>**GetDeskInfoList**</u>
- C2S

  ```protobuf
  message C2S_GetDeskInfoList {
  	sint32 game_id;		//	游戏ID, ID对应游戏通过读表同步
  }
  ```
- S2C

  ```protobuf
  message DeskInfo {
  	sint32 game_id;				// 游戏ID, ID对应游戏通过读表同步
  	sint32 desk_id;			    // 牌桌ID
  	sint32 player_number;	    // 牌桌内的玩家数量	
  	repeated string road_list;	// 根据游戏ID来，返回 不同类型的 路单
  }
  
  //	Successful Response
  message S2C_GetDeskInfoList {
  	repeated DeskInfo desk_info_list;
  }
  
  //	Fail Response
  message S2C_GetDeskInfoList {
  }
  ```

### 路单

#### 百家乐

```protobuf
//	庄: R
//	和: G
//	闲: B
//	庄对闲对: a
//	庄对: r
//	闲对: b
//	无对: n
//	天牌: 0 & 1
//	点数: 0-9
//	例子: 庄 + 9点 + 庄对 + 是天牌 = R9r1
message BaccaratRoad {
	string road;
}
```

#### 龙虎斗

```protobuf
//	虎: R
//	和: G
//	龙: B
//	点数: 0-9
//	例子: 龙 + 9点 = B9
message TheBigBattleRoad {
	string road;
}
```

#### 红黑大战

```protobuf
//	红: R
//	黑: B
//	黑皇: 1
//	红后: 2
//	豹子: 3
//	顺金: 4
//	金花: 5
//	顺子: 6
//	对子: 7
//	单张: 8
//	例子: 红 + 顺子 = R6
message RedBlackRoad {
	string road;
}
```

### 大厅路单更新

+ Type: Push

+ Topic: **<u>RefreshRoad</u>**

+ Push

  ```protobuf
  message Push_RefreshRoad {
  	sint32 game_id;				// 游戏ID, ID对应游戏通过读表同步
  	sint32 desk_id;			    // 牌桌ID
  	repeated string road_list;	// 根据游戏ID来，返回 不同类型的 路单
  }
  ```



### 获取牌局历史记录详细信息

+ Type: Requests and Responses

+ Topic: <u>**GetRoadDetailInfo**</u>

+ C2S

  ```protobuf
  message C2S_GetRoadDetailInfo {
  	string shoe_id;
  }
  ```

+ S2C

  ```protobuf
  // 经典百家乐
  message BaccaratDetailInfo {
  	int32 round_id;  				//	用于取值验证，最好唯一标识
  	sint32 player_point;			//	闲 点数
  	sint32 banker_point;			// 	庄 点数
  	repeated string player_cards;	// 	闲牌 详情
  	repeated string banker_cards;	// 	庄牌 详情
  	
  }
  
  //龙虎
  message TheBigBattleDetailInfo {
  	int32 round_id;  			//	用于取值验证，最好唯一标识符
  	string winner;				//	胜利者 dragon or tiger 待定
  	sint32 dragon_point;
  	sint32 tiger_point;
  	repeated Card dragon_card;
  	repeated Card tiger_card;
  }
  
  //红黑
  message RedBlackDetailInfo {
  	int32 round_id;  			//	用于取值验证，最好唯一标识符
  	string winner;				//	胜利者 red or black or 待定
  	sint32 red_point;     		//	红后点数（1-6 代表对子，单张，等等）
  	sint32 black_point;    		//	红后点数（1-6 代表对子，单张，等等）
  	repeated Card red_cards;
  	repeated Card black_cards;
  }
  
  message S2C_GetRoadDetailInfo {
  	sint32 game_count;			//	局数
  	sint64 shoeId_time;			//	牌靴产生的时间
  	sint64 curr_time;			//	当前时间 
  	repeated string road_list;	// 	路单
  	//	以下参数只存在其中一个
  	repeated BaccaratDetailInfo baccarat_list;    		//	百家乐路单, 参照 路单数据格式
  	
  	
  	repeated TheBigBattleDetailInfo theBigBattle_list;  //	龙虎斗路单, 参照 路单数据格式
  	repeated RedBlackDetailInfo redBlack_list;    		//	红黑大战路单, 参照 路单数据格式
  }
  ```



### 获取好路推荐数据

- Type: Requests and Responses

- Topic: <u>**GetGoodWayRecommend**</u>

- C2S

  ```protobuf
  message C2S_GetGoodWayRecommend {
  }
  ```

- S2C

  
  
  ```protobuf
  .message GoodWayInfo {
  	sint32 game_id;				// 	游戏ID, ID对应游戏通过读表同步
  	sint32 desk_id;				//	桌号
  	sint32 good_way_index;		//	好路类型, 读表
  	sint32 game_count;			//	局数
  	sint32 game_stage;			//	游戏阶段, 读表
  	sint32 begin_time;			//	阶段开始时间
  	sint32 end_time;			//	阶段结束时间
  	repeated string road_list;	// 	根据游戏ID来，返回 不同类型的 路单
  }
  
  message S2C_GetGoodWayRecommend {
  	repeated GoodWayInfo info_list;
  }
  ```



### 好路推荐数据更新

+ Type: Push

+ Topic: <u>**RefreshGoodWayRecommend**</u>

+ Push

  ```protobuf
  message Push_GetGoodWayRecommend {
  	repeated GoodWayInfo info_list;		//	是否同时推多条, 或者多次推送1条
  }
  ```

  

### 游戏流程

#### 通用的结构

```protobuf
//	花色:  Spade - 黑桃, Heart - 红桃, Diamond - 方块, Club - 梅花, Joker - 鬼牌
//	点数:  A = 1, J = 11, Q = 12, K = 13, 赖子 = 14, 大鬼 = 15, 数字 = 本身
//	组合方式:  花色小写的首字母 + 点数
//	示例:  s13 = 黑桃K, j15 = 大鬼, s14 = 赖子
message Card {
	string card;
}

//	百家乐、龙虎或红黑 下注区域
message BetInfo {
	sint32 player;			//	闲 -1：撤销全部
	sint32 banker;			//	庄 -1：撤销全部
	sint32 tie;				//	和 -1：撤销全部
	sint32 player_pair;		//	闲对 	-1：撤销全部 只有百家乐使用，龙虎和红黑不使用
	sint32 banker_pair;		//	庄对 	-1：撤销全部 只有百家乐使用，龙虎和红黑不使用
	sint32 lucky_six;		//	幸运6	-1：撤销全部 只有百家乐使用，龙虎和红黑不使用
}

//	百家乐、龙虎或红黑 下注区域 显示使用
message BetInfoShow {
	repeated sint32 player;			//	闲 
	repeated sint32 banker;			//	庄 
	repeated sint32 tie;			//	和 
	repeated sint32 player_pair;	//	闲对 	
	repeated sint32 banker_pair;	//	庄对 	
	repeated sint32 lucky_six;		//	幸运6
}

//	桌子上的玩家信息
message PlayerInfo {
	string user_id;			// 	玩家ID
	string name;			// 	玩家名字
	string portrait;		//	玩家头像 (URL)
	sint32 vip;				// 	是否是VIP
	sint64 chip;			// 	余额 (预留字段, 回执内不一定存在)	
	sint32 rank;			// 	龙虎, 红黑 排名
	BetInfoShow bet_list;	// 	百家乐、龙虎或红黑 下注信息
}
```



#### 百家乐

##### 进入牌桌

- Type: Requests and Responses

- Topic: **<u>NormalBaccaratEnterDesk</u>**

- C2S

  ```protobuf
  message C2S_EnterDesk {
  	sint32 desk_id;		//	1 - 99  ：指定进入牌桌, 0：则是快速加入 
  }
  ```

- S2C

  ```protobuf
  message S2C_EnterDesk {
  	sint32 desk_id;					//	桌号
  	sint32 limit_min;				//	限红下限
  	sint32 limit_max;				//	限红上限
  	string shoe_id;					//	牌靴号
  	repeated string all_card_list;	//	当前牌靴所有牌
  	repeated string used_cards;		//	已使用过的牌组
  	repeated PlayerInfo player_list;// 	桌子上玩家信息
  	sint32 game_count;				//	当前局数
  	Push_StageSync stage;       	//	当前阶段的数据
  	BetInfoShow bet_list			// 	已经下注的信息
  }
  ```

##### 进入牌桌推送

+ Type: Push

+ Topic: **NormalBaccaratEnterDeskSync**

  ```protobuf
  message Push_EnterDesk {
  	PlayerInfo player_list;			//	进入的 桌子的玩家信息
  }
  ```

##### 游戏阶段同步推送
+ Type: Push
+ Topic: <u>**NormalBaccaratStageSync**</u>
    ```protobuf
    //	下注阶段
    message BetStage {
        string lock_id;					//	预锁定的ID 目前没用，区块链使用
        repeated string all_card_list;	//	当前牌靴剩余的牌
        repeated string used_cards;		//	已使用过的牌组
    }

	//取数数据
    message OpenBlocks {
        string time;				//	区块链时间 毫秒
        string id;					//	区块链ID
        string hash;				//	区块链哈希值
		sint32 code;				//	区块链取数得到的值
    }
    //	取数阶段
    message FetchNumStage {
       repeated OpenBlocks open_blocks; 	//	锁定的取数数据
    }

    //开牌阶段
    message OpenCardStage {
        repeated string playerPoker;		//	闲家的牌
        repeated string bankerPoker;		//	庄家的牌
		bool isRed;							//	是否有红牌
    	 repeated OpenBlocks open_blocks; 	//	锁定的取数数据
}
    
    //	下注和结算 详细数据
    message SettleCardInfo {
        string userId;			//	玩家ID
       	BetInfoShowo betInfo; 	//	下注信息
        BetInfo settleInfo;		//	结算信息
    	sint64 chip;			// 	余额
    }
    
    //	结算阶段
    message SettleStage { 
        sint32 player_point;					//	闲家点数
	    sint32 banker_point;					//	庄家点数
    	BetInfo win_area;						//  赢的区域 1：赢 0：未赢
        repeated string playerPoker;			//	闲家的牌
        repeated string bankerPoker;			//	庄家的牌
        repeated SettleCardInfo settle_cards;	//	下注和结算 详细数据
    }
    
    //	换牌靴阶段
    message ShuffleStage {
        string shoe_id;
        repeated string card_list;
    }
        
    //	第一次订阅频道, 初始化同步信息
    //	切换游戏阶段
    message Push_StageSync {
        sint32 game_stage;		//	读表获取阶段ID
        sint32 game_count;		//	当前局数
        sint64 start_time;		//	开始时间
        sint64 ent_time;		//	结束时间
        sint64 push_time;		//	协议发出时间
        //	下列参数, 同时只存在一个
        BetStage bet_data;					//	下注阶段
        FetchNumStage FetchNum;				//	取数
        OpenCardStage open_card_data;		//	取数和开牌阶段
        SettleStage settle_data;			//	下注阶段
        ShuffleStage shuffle_data;			//	换牌靴阶段
    }
    ```

##### 下注阶段

###### 确认下注

-  Type: Requests and Push 

- Topic: **NormalBaccaratNotarizeBet**

- C2S

  ```protobuf
  //	确认下注
  message C2S_NotarizeBet {
		BetInfoShow bet_list;	//	发送下注的数额
  }
  ```

- S2C

  ```protobuf
  //	Successful Response
  message S2C_NotarizeBet {
  	BetInfoShow bet_list;	//	成功下注的数额
  	sint64 chip;			//	筹码余额  
  }
  ```
###### 撤销下注

-  Type: Requests and Push 

-  Topic: **NormalBaccaratRevocationBet**

- C2S

  ```protobuf
  //	确认下注
  message C2S_RevocationBet {
  	BetInfo bet_info;	//	发送下注的数额
  }
  ```

-  S2C

   ```prot
   //	Successful Response
   message S2C_RevocationBet {
   	sint64 chip;		//	筹码余额  
   }
   ```
###### 下注庄闲互换

+ Type: Requests and Push

+ Topic: **NormalBaccaratChangePB**

+ C2S

  ```protobuf
  //	庄闲互换
  message C2S_ChangePB {
  }
  ```

+ S2c

    ```protobuf
    //	Successful Response
    message S2C_ChangePB {
    }
    ```


##### 下注推送
###### 确认下注推送 
+ Type: Push

+ Topic: <u>**NormalBaccaratSyncBet**</u>

  ```protobuf
  message Push_BetSync {
		string user_id;			//	玩家Id
  	BetInfoShow bet_list;	//	获取对应位置所有的数额
  	sint64 chip;			//	当前的余额
  }
  ```
###### 庄闲互换推送

+ Type: Push

+ Topic: **NormalBaccaratSyncBetChangePB**

  ```protobuf
  message Push_BetSync
	```
  
######  撤销下注推送

+ Type: Push

+ Topic: **NormalBaccaratSyncRevocationBet**

  ```protobuf
  message Push_BetRevocationSync {
		string user_id;		//	玩家Id
  	BetInfo bet_info;	//	获取对应 位置
  	sint64 chip;		//	当前的余额
  }
  ```


##### 离开牌桌

- Type: Requests and Push

- Topic: **NormalBaccaratLeaveDesk**

- C2S

  ```protobuf
  message C2S_LeaveDesk {
  }
  ```

- S2C

  ```protobuf
  message S2C_LeaveDesk {
  }	
  ```

##### 离开牌桌 推送

+ Type: Push

+ Topic: **NormalBaccaratLeaveDeskSync**

  ```protobuf
  message Push_LeaveDesk {
  	string user_id;			//	玩家Id
  }
  ```


#### 龙虎斗

##### 进入牌桌

- Type: Requests and Responses

- Topic: **<u>TheBigBattleEnterDesk</u>**

- C2S

  ```protobuf
  message C2S_EnterDesk {
  	sint32 desk_id;		//	可选字段, 填写牌桌ID指定进入牌桌, 不填写则是快速加入
  }
  ```

- S2C

  ```protobuf
  message S2C_EnterDesk {
  	sint32 desk_id;		//	桌号
  	sint32 limit_min;	//	限红下限
  	sint32 limit_max;	//	限红上限
  	sint32 shoe_id;		//	牌靴号
  	sint32 enter_time;	//	进入牌桌时间, 以服务器时间为准
  	repeated PlayerInfo player_list;
  	repeated Card all_card_list;	//	当前牌靴所有牌
  	repeated Card using_cards;		//	选定后, 使用中的牌组
  	repeated Card used_cards;		//	已使用过的牌组
  }
  ```



##### 活跃玩家列表

+ Type: Push

+ Topic: **<u>TheBigBattleActiveList</u>**

  ```protobuf
  message ActivePlayer {
  	string name;	//	玩家名字
  	sint32 balance;	//	玩家余额
  	string rank;	//	玩家列表位置
  }
  
  //	第一次订阅频道, 初始化同步信息
  //	变动值发送, 每次按照游戏类型发送对应的数量
  message Push_ActiveList {
  	repeated ActivePlayer active_list;
  }
  ```

  

##### 游戏阶段同步, 切换

- Type: Push

- Topic: <u>**TheBigBattleStageSync**</u>

  ```protobuf
  //	下注阶段
  message BetStage {
  	string lock_id;					//	预锁定的ID
  	sint32 used_card_number;			//	已用牌数
      sint32 all_card_number;			//	总牌数
  	repeated Card using_cards;		//	选定后, 使用中的牌组
  }
  
  //	开牌阶段
  message ResultInfo {
  	string time;				//	区块链时间
  	sint32 id;					//	区块链ID
  	string hash;				//	区块链哈希值
  }
  
  message OpenCardStage {
  	repeated ResultInfo result_list;
  }
  
  //	结算阶段
  message SettleCardInfo {
  	string owner;
  	string card;
  }
  
  message SettleStage {
  	sint32 dragon_point;
  	sint32 tiger_point;
  	repeated TheBigBattleBetInfo settle_list;			//	牌桌内每个人的下注信息
  	repeated SettleCardInfo settle_cards;				//	已发到牌桌上的牌
  }
  
  //	换牌靴阶段
  message ShuffleStage {
  	sint32 shoe_id;					//	牌靴
  	repeated Card card_list;		//	新的全部牌
  }
  
  //	第一次订阅频道, 初始化同步信息
  //	切换游戏阶段
  message Push_StageSync {
  	sint32 game_stage;		//	读表获取阶段ID
  	sint32 push_time;		//	协议发出时间
  	sint32 game_count;		//	当前局数
  	//	下列参数, 同时只存在一个
  	BetStage bet_data;
  	OpenCardStage open_card_data;
  	SettleStage settle_data;
  	ShuffleStage shuffle_data;
  }
  ```



##### 确认, 撤销下注

- Type: Requests and Push

- Topic: <u>**TheBigBattleSyncBet**</u>

- C2S

  ```protobuf
  //	确认下注
  message C2S_NotarizeBet {
  	TheBigBattleBetInfo bet_info;		//	发送下注的数额
  }
  
  //	庄闲互换
  message C2S_ChangeDT {
  }
  
  //	撤销下注
  message C2S_RevocationBet {
  	TheBigBattleBetInfo bet_info;		//	发送撤销的数额
  }
  ```

- Push

  ```protobuf
  message Push_BetSync {
  	TheBigBattleBetInfo bet_info;		//	获取对应位置所有的数额
  }
  ```



##### 离开牌桌

- Type: Requests and Push

- Topic: <u>**TheBigBattleLeaveDesk**</u>

- C2S

  ```protobuf
  message C2S_LeaveDesk {
  }
  ```

- S2C

  ```protobuf
  message S2C_LeaveDesk {
  }
  ```



#### 红黑大战

##### 进入牌桌

+ Type: Requests and Responses

+ Topic: **<u>RedBlackEnterDesk</u>**

+ C2S

  ```protobuf
  message C2S_EnterDesk {
  	sint32 desk_id;		//	可选字段, 填写牌桌ID指定进入牌桌, 不填写则是快速加入
  }
  ```

+ S2C

  ```protobuf
  message S2C_EnterDesk {
  	sint32 desk_id;					//	桌号
  	sint32 limit_min;				//	限红下限
  	sint32 limit_max;				//	限红上限
  	sint32 shoe_id;					//	牌靴号
  	sint32 enter_time;				//	进入牌桌时间, 以服务器时间为准
  	repeated PlayerInfo player_list;
  	repeated Card using_cards;		//	选定后, 使用中的牌组
  }
  ```



##### 活跃玩家列表

- Type: Push

- Topic: **<u>RedBlackActiveList</u>**

  ```protobuf
  message ActivePlayer {
  	string name;	//	玩家名字
  	sint32 balance;	//	玩家余额
  	string rank;	//	玩家列表位置
  }
  
  //	第一次订阅频道, 初始化同步信息
  //	变动值发送, 每次按照游戏类型发送对应的数量
  message Push_ActiveList {
  	repeated ActivePlayer active_list;
  }
  ```

  

##### 游戏阶段同步, 切换

- Type: Push

- Topic: <u>**RedBlackStageSync**</u>

  ```protobuf
  //	下注阶段
  message BetStage {
  	string lock_id;					//	预锁定的ID
  	repeated Card using_cards;		//	选定后, 使用中的牌组
  }
  
  //	开牌阶段
  message ResultInfo {
  	string time;				//	区块链时间
  	sint32 id;					//	区块链ID
  	string hash;				//	区块链哈希值
  }
  
  message OpenCardStage {
  	repeated ResultInfo result_list;
  }
  
  //	结算阶段
  message SettleCardInfo {
  	string owner;
  	string card;
  }
  
  message SettleStage {
  	string red_type;						//	读表获取类型
  	string black_type;						//	读表获取类型
  	string winner;							//	red or black
      repeated RedBlackBetInfo settle_list;	//	牌桌内每个人的下注信息
  	repeated SettleCardInfo settle_cards;	//	已发到牌桌上的牌
  }
  
  //	第一次订阅频道, 初始化同步信息
  //	切换游戏阶段
  message Push_StageSync {
  	sint32 game_stage;		//	读表获取阶段ID
  	sint32 push_time;		//	协议发出时间
  	sint32 game_count;		//	当前局数
  	//	下列参数, 同时只存在一个
  	BetStage bet_data;
  	OpenCardStage open_card_data;
  	SettleStage settle_data;
  }
  ```



##### 确认, 撤销下注

- Type: Requests and Push

- Topic: <u>**RedBlackSyncBet**</u>

- C2S

  ```protobuf
  //	确认下注
  message C2S_NotarizeBet {
  	RedBlackBetInfo bet_info;		//	发送下注的数额
  }
  
  //	庄闲互换
  message C2S_ChangeRB {
  }
  
  //	撤销下注
  message C2S_RevocationBet {
  	RedBlackBetInfo bet_info;		//	发送撤销的数额
  }
  ```

- Push

  ```protobuf
  message Push_BetSync {
  	RedBlackBetInfo bet_info;		//	获取对应位置所有的数额
  }
  ```




##### 离开牌桌

- Type: Requests and Push

- Topic: <u>**RedBlackLeaveDesk**</u>

- C2S

  ```protobuf
  message C2S_LeaveDesk {
  }
  ```

- Push

  ```protobuf
  message S2C_LeaveDesk {
  }
  ```




#### 幸运大转轮

##### 进入游戏

+ Type: Requests and Responses

+ Topic: **<u>LuckWeelEnterGame</u>**

+ C2S

  ```protobuf
  message C2S_LuckWeelEnterGame {
  }
  ```

+ S2C

  ```protobuf
  message S2C_LuckWeelEnterGame {
  }
  ```

  

##### 确认, 撤销下注

- Type: Requests and Responses

- Topic: <u>**LuckWeelBet**</u>

- Struct

  ```protobuf
  message LuckWeelBetInfo {
  	sint32 diamond_1;
  	sint32 club_3;
  	sint32 heart_7;
  	sint32 spade_10;
  	sint32 jack_20;
  	sint32 queen_45;
  	sint32 king_45;
  }
  ```

- C2S

  ```protobuf
  //	确认下注
  message C2S_LuckWeelBet {
  	LuckWeelBetInfo bet_info;		//	发送下注的数额
  }
  
  //	撤销下注
  message C2S_LuckWeelRevocationBet {
  	LuckWeelBetInfo bet_info;		//	发送撤销的数额
  }
  ```

- S2C

  ```protobuf
  message S2C_LuckWeelBet {
  	LuckWeelBetInfo bet_info;		//	获取对应位置所有的数额
  }
  ```



#### 蛇神之环

##### 蛇神之环下注

- Type: Push
- Topic: <u>**SnakeWeelBet**</u>

```protobuf
message Snake_BetInfo{
     sint32 rank; //下注者所在排名位置
     string userid //玩家id
     uint32 bet_num //下注数量   
     
}

message Push_SnakeWeelBet { 
	BetInfo bet_info_list;		//	下注列表
}
```





## 多台百家乐

### 获取多台百家乐列表（12桌）

- Type:Requests and Responses

- Topic:<u>**GetMoreBaccaratDeskInfoList**</u>

- C2S

  ```protobuf
  message C2S_GetMoreBaccaratDeskInfoList{
   	
  }
  ```

- S2C

  ```protobuf
  message MoreBaccaratDeskInfo{
  	sint32 game_id;			//游戏ID,ID对应游戏通过读表同步
  	sint32 desk_id;			//牌桌ID
  	string hero_icon;		//人物头像地址（图片可能存储在本地，可能为头像名称）
  	
  	repeated string road_list;		//百家乐路单格式
  	
  	//Push_StageSync 百家乐游戏阶段同步（下注，开牌，结算，洗牌）
  	sint32 game_stage;		//	读表获取阶段ID
  	sint64 start_time;		//	开始时间
      sint64 ent_time;		//	结束时间
      sint64 push_time;		//	协议发出时间
      //	下列参数, 同时只存在一个
      BetStage bet_data;					//	下注阶段
      OpenCardStage open_card_data;		//	取数和开牌阶段
      SettleStage settle_data;			//	结算阶段
      ShuffleStage shuffle_data;			//	换牌靴阶段
  }
  
  //Successful Response
  message S2C_GetMoreBaccaratDeskInfoList{
  	repeated MoreBaccaratDeskInfo desk_info_list;
  }
  //Fail Response
  message S2C_GetMoreBaccaratDeskInfoList{
  
  }
  ```



### 多台列表刷新（12桌）

- Type:Push 

- Topic:<u>**RefreshMoreBaccarat**</u>

- Push

  ```protobuf
  message Push_RefreshMoreBaccarat{
  	sint32 desk_id;
  	
  	//repeated string road_list;		//百家乐路单格式,结算状态时有值
  	//Push_StageSync 百家乐游戏阶段同步（下注，开牌，结算，洗牌）
  	sint32 game_stage;		//	读表获取阶段ID
  	sint64 start_time;		//	开始时间
      sint64 ent_time;		//	结束时间
      sint64 push_time;		//	协议发出时间
       //	下列参数, 同时只存在一个,只需简化版
      BetStage bet_data;					//	下注阶段
      OpenCardStage open_card_data;		//	取数和开牌阶段
      SettleStage settle_data;			//	结算阶段,数据没错可以从此得到单个路单数据（待定）
      ShuffleStage shuffle_data;			//	换牌靴阶段
  }
  ```

  



### 获取多台百家乐所有桌子列表

- Type:Requests and Responses

- Topic:<u>**GetMoreBaccaratAllDeskInfoList**</u>

- C2S

  ```protobuf
  message C2S_GetMoreBaccaratAllDeskInfoList{
  	
  }
  
  
  ```

- S2C

  ```protobuf
  message MoreBaccaratListDeskInfo{
  	sint32 game_id;			//游戏ID,ID对应游戏通过读表同步
  	sint32 desk_id;			//牌桌ID
  	string hero_icon;		//人物头像地址（图片可能存储在本地，可能为头像名称）
  	
  	sint32 good_way_type;	//好路类型 0表示不是好路
  	repeated string road_list;		//百家乐路单格式
  	
  	//Push_StageSync 百家乐游戏阶段同步（下注，开牌，结算，洗牌）
  	sint32 game_stage;		//	读表获取阶段ID
  	sint64 start_time;		//	开始时间
      sint64 ent_time;		//	结束时间
      sint64 push_time;		//	协议发出时间
  }
  
  message S2C_GetMoreBaccaratAllDeskInfoList{
  	repeated MoreBaccaratListDeskInfo desk_info_list;
  }
  
  ```



### 多台桌子列表刷新

- Type:Push

- Topic:<u>**RefreshMoreBaccaratList**</u>

- Push

  ```protobuf
  message Push_RefreshMoreBaccaratList{
  	sint32 desk_id;
  	
  	sint32 good_way_type;	//好路类型 0表示不是好路
  	string road;		//百家乐路单格式,结算时刷新
  	
  	//Push_StageSync 百家乐游戏阶段同步（下注，开牌，结算，洗牌）
  	sint32 game_stage;		//	读表获取阶段ID
  	
  }
  ```

### 多台交换桌子

- Type:Requests and Respones

- Topic:<u>**MoreBaccaratExchangeDesk**</u>

- C2S

  ```protobuf
  message C2S_MoreBaccaratExchangeDesk{
  	sint32 old_game_id;
  	sint32 old_desk_id;
  	sint32 new_game_id;
  	sint32 new_desk_id;
  }
  ```

- S2C

  ```protobuf
  message S2C_MoreBaccaratExchangeDesk{
  	MoreBaccaratDeskInfo new_desk_info;		//新桌子信息
  	MoreBaccaratListDeskInfo old_desk_info;		//原桌子被替换后的信息（待定）
  }
  ```



### 历史记录（玩家输赢记录）

- Type:Requests and Responses

- Topic:<u>**GetChipHistoryRecordList**</u>

- C2S

  ```protobuf
  //<策划案/百家乐/历史记录>
  message C2S_GetChipHistoryRecordList{
  	sint32 click_type;		//1:投注记录 2：额度记录
  	sint32 time_type;		//1:一天内，2：3天内；3：1周内；4：1个月
  	sint32 game_id;		//0：全部,其余为游戏配表id
  }
  ```

- S2C

  ```protobuf
  message ChipInfo{		//投注记录
  	string round_id;		//局号
  	sint32 time;			//该局开始的时间
  	sint32 game_id;
  	sint32 desk_id;
  	string result; //player-point,banker-point
  	string play_type;		//玩法 （百家乐有 经典-幸运6），其他游戏需查看策划案《历史记录》
  	uint64 play_money;		//该局下注总额
  	int64 win_money;		//下注输赢结果	
  }
  
  message LimitInfo{	//额度记录
  	sint32 time;
  	bool bWin;	//下注是否赢钱
  	uint64 before_money;		//下注前余额
  	int64 income_moeny;		//收入
  	int64 expend_money;		//下注总额度
  	uint64 after_money;		//下注后金额
  }
  
  message S2C_GetChipHistoryRecordList{
  	sint32 click_type;		//显示哪种类型记录
  	uint64 total_play_money;		//查询时间段内下注总额(收入)
  	int64 total_win_money;			//查询时间段内输赢总额（支出）
  	
  	//click_type 不同，二选一
  	repeated ChipInfo chip_info_list;
  	repeated LimitInfo limit_info_list;
  }
  ```



### 复核过程

- Type:Requests and Responses

- Topic:GetReCheckInfo

- C2S

  ```protobuf
  message C2S_GetReCheckInfo{
  	sint32 game_id;
  	string round_id;
  }
  ```

- S2C

  ```protobuf
  message CommonRoundInfo{
  	sint32 player_point;
  	sint32 banker_point;
  	repeated Card card_list;
  }
  
  message VideoPokerInfo{
  	repeated Card card_list;
  }
  
  message LuckyWheelInfo{
  	repeated string num_list;
  }
  
  message S2C_GetRecheckInfo{
  	sint32 game_id;			//用于游戏区分
  	string round_id;		//局号
  	sint32 pre_time;		//预定EOS时间
  	string pre_eos_num;			//预定EOS区块号
  	sint32 time;			//Eos产生时间
  	string eos_num;			//EOS
  	
  	//甴game_id区别，只有一个被赋值
  	CommonRoundInfo common_round_info;
  	VideoPokerInfo video_poker_info;
  	LuckyWheelInfo lucky_wheel_info;
  }
  ```

  

### 刷新玩家筹码

- Type:Push 

- Topic:<u>**RefreshPlayerChip**</u>

- Push

  ```protobuf
  message Push_RefreshPlayerChip{
  	sint32 type;			//1:筹码 2：积分
  	sint32 dynamic_money;	//变动的金额
  	sint32 total_money;		//变动后的总金额
  }
  ```



