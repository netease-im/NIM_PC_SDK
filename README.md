#Windows(PC) SDK 开发手册

## <span id="网易云信服务概要">网易云信服务概要</span>
云信以提供客户端SDK（覆盖Android、iOS、Web、PC）和服务端OPEN API 的形式提供即时通讯云服务。开发者只需想好APP 的创意及UI 展现方案，根据APP 业务需求选择云信的相应功能接入即可。

## <span id="开发准备">开发准备</span>
###<span id="总体接口约定">总体接口约定</span>
SDK 提供的所有API 都是C接口，根据模块划分为不同的API 文件，只提供动态加载dll 的方式获取API 并调用。App 开发者只需要引用SDK包里nim\_include 目录下的define 子目录里的模块定义头文件（命名方式如nim\_xxx\_def.h）即可。关于API 的定义，可以查看API 文档或SDK 包里nim\_include目录下的API 子目录里的API 头文件（命名方式如nim\_xxx.h）。一般来说，每个模块都有对应的API 头文件和定义头文件。

SDK 提供了2种类型的接口。

第一种：注册回调和执行接口分开定义，这类接口需要提前注册好回调函数，然后执行接口时，调用相应的回调函数输出结果，App 上层需要在回调函数里处理结果。这类回调函数可能由调用执行接口触发，也有可能由SDK 主动触发，一般由SDK 主动触发回调函数（如接收消息等）。

第二种：回调函数作为参数，传入执行接口，然后执行接口时，会触发传入的回调函数。

###<span id="SDK数据目录">SDK数据目录</span>
当收到多媒体消息后，SDK 会负责下载这些多媒体文件，同时SDK 还要记录一些log，因此SDK 需要一个数据缓存目录。该目录由第三方App 通过nim\_client\_init 初始化接口传入，默认为存放到系统的AppData 目录下，App 传入一个目录名即可，SDK 会自动生成用户数据缓存目录。数据缓存目录默认为"{系统的AppData 目录}\\{App 传入的目录名}\\NIM\\{某个用户对应的用户数据目录}”，还可以由App 完全自定义用户数据目录，需要传入完整的路径，并确保读写权限正确。如果第三方App 需要清除缓存功能，可扫描该目录下的文件，按照你们的规则清理即可。
具体某个用户对应的缓存目录下面包含如下子目录：

- image：图片消息文件
- audio：语音消息文件
- video：视频消息文件
- res：其他资源文件

SDK 提供了接口nim\_tool\_get\_user\_specific\_appdata\_dir 获取某个用户对应的具体类型的App data 目录（如图片消息文件存放目录，语音消息文件存放目录等）（注意：需要调用nim\_free\_buf 接口释放其返回的内存）。 

## <span id="初始化SDK">初始化SDK</span>
准备工作：将SDK 相关的dll 文件（nim.dll，nim\_audio.dll, nim\_tools\_http.dll, nim\_videochat.dll）放到App 的运行目录下，并将SDK 的配置文件目录nim\_conf（目录里包含一个ver\_ctrl.dll文件）放到App 的运行目录下。SDK 基于vs2010 开发，如果App 没有对应的运行时库文件（msvcp100.dll和msvcr100.dll），请将其放到App 的运行目录下。

准备工作完成后，在程序启动时，调用LoadLibrary 函数动态加载nim.dll，然后调用GetProcAddress获取API 接口：nim\_client\_init，调用此接口初始化NIM SDK， 同时，SDK 能力的一些参数以及如果SDK 需要连接独立部署的服务器的地址等配置也是在初始化SDK 时传入。

	typedef bool(*nim_client_init)(const char *app_data_dir, const char *app_install_dir, const char *json_extension);

	void foo()
	{
		//获取SDK初始化接口
		HINSTANCE hInst = LoadLibraryW(L"nim.dll");
		nim_client_init func = (nim_client_init) GetProcAddress(hInst, "nim_client_init");

		//组装SDK初始化参数
		Json::Value config_root;
	
		//组装SDK能力参数（必填）
		Json::Value config_values;
		config_values[kNIMDataBaseEncryptKey] = ""; //string（db key必填，目前只支持最多32个字符的加密密钥！建议使用32个字符）
		config_values[kNIMPreloadAttach] = ;		//bool （选填，是否需要预下载附件缩略图， SDK默认预下载）
		config_root[kNIMGlobalConfig] = config_values;
	
		//组装SDK独立部署的服务器配置（选填）
		Json::Value server_values;
		server_values[kNIMLbsAddress] = "http://xxx"; 			//lbs地址
		server_values[kNIMNosLbsAddress] = "http://xxx";		//nos lbs地址
		server_values[kNIMDefaultLinkAddress].append("xxx.xxx.xxx.xxx:xxxx");				//默认的link地址
		server_values[kNIMDefaultNosUploadAddress].append("http://xxx.xxx.xxx.xxx:xxxx"); 	//默认的nos上传地址
		server_values[kNIMDefaultNosDownloadAddress].append("http://xxx.xxx.xxx.xxx:xxxx");	//默认的nos下载地址
		server_values[kNIMRsaPublicKeyModule] = "";				//密钥
		server_values[kNIMRsaVersion] = 0;						//密钥版本号  	
		config_root[kNIMPrivateServerSetting] = server_values;

		//初始化SDK
		func("appdata path", "app installation path", config_root.toStyledString().c_str());
	}

	
在程序退出前，调用接口nim\_client\_cleanup 进行NIM SDK 的退出工作，然后调用FreeLibrary 函数释放nim.dll。
	
	typedef	void(*nim_client_cleanup)(const char *json_extension);

	void foo()
	{
		nim_client_cleanup func = (nim_client_cleanup) GetProcAddress(hInst, "nim_client_cleanup");
		func("");
		FreeLibrary(hInst);
	}

##登录与登出

###<span id="首次登录">首次登录</span>
  
  SDK 登录后会同步群信息，离线消息，漫游消息，系统通知等数据，所以首次登录前需要提前注册一些全局回调（具体说明请参阅API 文档），以下以注册最近会话变更通知回调为例：

	//全局会话列表变更通知函数
	void CallbackSessionChange(int rescode, const char *result, int total_unread_counts, const char *json_extension, const void *user_data)
	{
		if (rescode == kNIMResSuccess)
		{
			Json::Value value;
			Json::Reader reader;
			if (reader.parse(result, value))
			{
				nim::NIMSessionCommand command = nim::NIMSessionCommand(value[nim::kNIMSessionCommand].asInt());
				switch (command)
				{
					case kNIMSessionCommandAdd:
						...
					case kNIMSessionCommandUpdate:	
						...
					...				
				}

			}
		}
		else
		{
			...
		}
	}
	
	typedef void(*nim_session_reg_change_cb)(const char *json_extension, nim_session_change_cb_func cb, const void *user_data);

	void foo()
	{
		//注册全局会话列表变更通知函数
		nim_session_reg_change_cb func = (nim_session_reg_change_cb)GetProcAddress(hInst, "nim_session_reg_change_cb");
		func(nullptr, &CallbackSessionChange, nullptr);	// 会话列表变更通知(nim_session)
	}


同时还有以下全局回调（具体说明请参阅API 文档）：

	void nim_data_sync_reg_complete_cb(nim_data_sync_cb_func cb, const void *user_data);			// 数据同步结果通知(nim_data_sync)
	void nim_talk_reg_receive_cb(const char *json_extension, nim_talk_receive_cb_func cb, const void *user_data);		// 接收会话消息通知(nim_talk)
	void nim_sysmsg_reg_sysmsg_cb(const char *json_extension, nim_sysmsg_receive_cb_func cb, const void *user_data);	// 接收系统消息通知(nim_sysmsg)
	void nim_talk_reg_arc_cb(const char *json_extension, nim_talk_arc_cb_func cb, const void *user_data);			// 发送消息结果通知(nim_talk)
	void nim_talk_reg_custom_sysmsg_arc_cb(const char *json_extension, nim_custom_sysmsg_arc_cb_func cb, const void *user_data);// 发送自定义系统通知的结果通知(nim_talk)
	void nim_team_reg_team_event_cb(const char *json_extension, nim_team_event_cb_func cb, const void *user_data);	// 群组事件通知(nim_team)
	void nim_client_reg_kickout_cb(const char *json_extension, nim_json_transport_cb_func cb, const void *user_data);		// 帐号被踢通知(nim_client)
	void nim_client_reg_disconnect_cb(const char *json_extension, nim_json_transport_cb_func cb, const void *user_data);	// 网络连接断开通知(nim_client)
	void nim_client_reg_kickout_other_client_cb(const char *json_extension, nim_json_transport_cb_func cb, const void *user_data); // 将本帐号的其他端踢下线的结果通知(nim_client)
	void nim_friend_reg_changed_cb(const char *json_extension, nim_friend_change_cb_func cb, const void *user_data);	// 好友通知 
	void nim_user_reg_special_relationship_changed_cb(const char *json_extension, nim_user_special_relationship_change_cb_func cb, const void *user_data);	// 用户特殊关系通知

登录

	void CallbackLogin(const char* res, const void *user_data)
	{
		Json::Value values;
		Json::Reader reader;
		if (reader.parse(res, values) && values.isObject())
		{
			if (values[kNIMLoginStep].asInt() == kNIMLoginStepLogin && values[kNIMErrorCode].asInt() == 200)
			{
				...
			}
			...
		}
	}

	typedef	void(*nim_client_login)(const char *app_token, const char *account, const char *password, const char *json_extension, nim_json_transport_cb_func cb, const void* user_data);

	void foo()
	{
		nim_client_login func = (nim_client_login) GetProcAddress(hInst, "nim_client_login");
		func("app key", "app account", "token", nullptr, &CallbackLogin, nullptr);
		//app key: 应用标识，不同应用之间的数据（用户，消息，群组等）是完全隔离的。开发自己的应用时，需要替换成自己申请来的app key
	}

###<span id="自动重新登录">自动重新登录</span>
  
  SDK 在网络连接断开后，会监听网络状况，如果网络状况好转，那么SDK 会进行重新登录，登录结果可以在注册的重新登录回调函数中处理。

###<span id="手动重新登录">手动重新登录</span>

  SDK 在重新登录失败后，可以由用户手动调用重新登录接口nim\_client\_relogin。

###<span id="登出">登出</span>
  
  执行接口nim\_client\_logout 进行退出，必须在退出回调函数中退出主程序，在退出之前，主程序可以进行退出前的一些操作。

	void CallbackLogout(const char* res, const void *user_data)
	{
		...
	}

	typedef	void(*nim_client_logout)(nim::NIMLogoutType logout_type, const char *json_extension, nim_json_transport_cb_func cb, const void* user_data);

	void foo()
	{
		nim_client_logout func = (nim_client_logout) GetProcAddress(hInst, "nim_client_logout");
		func(kNIMLogoutAppExit, nullptr, &CallbackLogout, nullptr);
	}

###<span id="断网重连通知">断网重连通知</span>
  
  通过接口nim\_client\_reg\_disconnect_cb 来注册连接断开回调函数，断开连接后根据实际需要做后续处理（比如提示用户）。

	void CallbackDisconnect(const char* res, const void* user_data)
	{
		//解析res
	}

	typedef void(*nim_client_reg_disconnect_cb)(const char *json_extension, nim_json_transport_cb_func cb, const void* user_data);

	void foo()
	{
		nim_client_reg_disconnect_cb func = (nim_client_reg_disconnect_cb) GetProcAddress(hInst, "nim_client_reg_disconnect_cb");
		func(nullptr, &CallbackDisconnect, nullptr);
	}
	

###<span id="被踢出通知">被踢出通知</span>
  
  通过接口nim\_client\_reg\_kickout_cb 来注册被踢出回调函数，当收到此通知时，一般是退出程序然后回到登录窗口，回调函数参数中详细说明了被踢出的原因。

	void CallbackKickout(const char* res, const void* user_data)
	{
		//解析res
	}

	typedef void(*nim_client_reg_kickout_cb)(const char *json_extension, nim_json_transport_cb_func cb, const void* user_data);

	void foo()
	{
		nim_client_reg_kickout_cb func = (nim_client_reg_kickout_cb) GetProcAddress(hInst, "nim_client_reg_kickout_cb");
		func(nullptr, &CallbackKickout, nullptr);
	}
	

## <span id="基础消息功能">基础消息功能</span>  
###<span id="消息功能概述">消息功能概述</span>
SDK(PC) 原生支持发送文本、图片、文件等3种类型消息，同时支持用户发送自定义的消息类型。

###<span id="发送消息">发送消息</span>

	void nim_talk_send_msg(const char *json_msg, const char *json_extension, nim_http_up_prg_cb_func prg_cb, const void *prg_user_data)

以发送图片消息的为例：.

	typedef void(*nim_talk_send_msg)(const char* json_msg, const char *json_extension, nim_http_up_prg_cb_func prg_cb, const void *prg_user_data);
	
	void foo()
	{
		Json::Value json;
		json[kNIMMsgKeyToType]			= ; //int 会话类型(NIMSessionType 个人0 群组1)
		json[kNIMMsgKeyToAccount]		= ; //string 消息接收者id
		json[kNIMMsgKeyTime]			= ; //int64  消息发送时间（毫秒），可选
		json[kNIMMsgKeyType]			= ; //NIMMessageType 消息类型
		json[kNIMMsgKeyBody]			= ; //string 消息内容
		json[kNIMMsgKeyAttach]			= ; //string 附加内容
		json[kNIMMsgKeyClientMsgid]		= ; //string 消息id，一般使用guid
		json[kNIMMsgKeyResendFlag]		= ; //int    重发标记(0不是重发, 1是重发)
		json[kNIMMsgKeyLocalLogStatus]	= ; //NIMMsgLogStatus 消息状态
		json[kNIMMsgKeyType] 			= kNIMMessageTypeImage;

		//图片详细信息
		Json::Value attach;
		attach[kNIMImgMsgKeyMd5] = ;  	//string 文件md5，选填
		attach[kNIMImgMsgKeySize] = ; 	//long   文件大小，选填
		attach[kNIMImgMsgKeyWidth] = ;  //int	 图片宽度，必填
		attach[kNIMImgMsgKeyHeight] = ; //int	 图片高度，必填
		json[kNIMMsgKeyAttach] = attach.toStyledString();

		json[kNIMMsgKeyLocalFilePath]	= ; //string	图片本地路径
		json[kNIMMsgKeyLocalLogStatus] = kNIMMsgLogStatusSending;

		nim_talk_send_msg func = (nim_talk_send_msg) GetProcAddress(hInst, "nim_talk_send_msg");
		func(json.toStyledString().c_str(), "", NULL, "");
	}

发送消息状态通过登录前注册的全局回调获取。

	void CallbackSendMsgResult(const char* result, const void *user_data)
	{
		//解析result {"msg_id" : "" , "talk_id" : "" , "rescode" : ""}
	}

	typedef void(*nim_talk_reg_arc_cb)(const char *json_extension, nim_talk_arc_cb_func cb, const void *user_data);

	void foo()
	{
		nim_talk_reg_arc_cb func = (nim_talk_reg_arc_cb) GetProcAddress(hInst, "nim_talk_reg_arc_cb");
		func(nullptr, &CallbackSendMsgResult, nullptr);
	}

如果要停止正在发送中的消息（目前只支持发送文件消息时的终止），可以调用nim\_talk\_stop\_send\_msg 接口实现。

###<span id="接收消息">接收消息</span>

提前注册好接收消息的回调函数。在接收到消息时，如果是图片、语音消息，那么SDK 会自动下载，然后通过注册的HTTP回调函数通知。如果下载失败，则调用接口nim\_http\_fetch\_media 重新下载。此外，还可以调用nim\_http\_stop\_fetch\_media 停止下载（目前仅对文件消息类型有效）。

	void CallbackReceiveMsg(const char* msg, const char *json_exten, const void *user_data)
	{
		Json::Value value;
		Json::Reader reader;
		if (reader.parse(msg, value))
		{
			int rescode = value[nim::kNIMMsgKeyLocalRescode].asInt();
			int feature = value[nim::kNIMMsgKeyLocalMsgFeature].asInt();

			switch (feature)
			{
			case kNIMMessageFeatureDefault:
				...
			case kNIMMessageFeatureSyncMsg:
				...
			...
			}
		}
	}

	typedef void(*nim_talk_reg_receive_cb)(const char *json_extension, nim_talk_receive_cb_func cb, const void* user_data);

	void foo()
	{
		nim_talk_reg_receive_cb func = (nim_talk_reg_receive_cb) GetProcAddress(hInst, "nim_talk_reg_receive_cb");
		func(nullptr, &CallbackReceiveMsg, nullptr);
	}

###<span id="最近会话">最近会话</span>

最近会话记录了与用户最近有过会话的联系人信息，包括联系人帐号、联系人类型、最近一条消息的时间、消息状态、消息缩略和未读条数等信息。

	void CallbackQuerySessionListCb(int total_unread_count, const char* session_list, const char *json_exten, const void *user_data)
	{
		...
	}

	typedef void(*nim_session_query_all_recent_session_async)(const char *json_extension, nim_session_query_recent_session_cb_func cb, const void* user_data);

	void foo()
	{
		nim_session_query_all_recent_session_async func = (nim_session_query_all_recent_session_async) GetProcAddress(hInst, "nim_session_query_all_recent_session_async");
		func(nullptr, &CallbackQuerySessionListCb, nullptr);
	}
	

在收发消息的同时，SDK会更新对应聊天对象的最近联系人资料。当有消息收发时，SDK 会发出最近联系人更新通知，使用nim\_session\_reg\_change\_cb 注册通知回调函数。

SDK 也提供了删除单个和删除全部最近联系人的接口。需要注意：删除会话项时本地和服务器会同步删除！

- 删除单个

		void nim_session_delete_recent_session_async(NIMSessionType to_type, const char *id, const char *json_extension, nim_session_change_cb_func cb, const void *user_data);

- 删除全部

		void nim_session_delete_all_recent_session_async(const char *json_extension, nim_session_change_cb_func cb, const void *user_data);

###<span id="自定义消息">自定义消息</span>

除了内建消息类型外，SDK 也支持收发自定义消息类型。和透传消息一样，SDK 也不会解析自定义消息的具体内容，解释工作由开发者完成。

## <span id="多媒体消息">多媒体消息</span>   

###<span id="初始化与清理">初始化与清理</span>

在程序启动时，调用LoadLibrary 加载nim\_audio.dll；然后调用接口nim\_audio\_init\_module 初始化语音模块；在退出程序前，先调用接口nim\_audio\_uninit\_module 释放语音模块，然后调用FreeLibrary 释放dll。

###<span id="语音播放与停止">语音播放与停止</span>
  
调用接口nim\_audio\_reg\_start\_play\_cb 注册语音播放开始回调函数，调用接口nim\_audio\_reg\_stop\_play\_cb 注册语音播放结束回调函数。调用接口nim\_audio\_play\_audio 播放语音。调用接口nim\_audio\_stop\_play\_audio 停止播放。

### <span id="语音转文字">语音转文字</span>
SDK 提供语音转文字接口。

	void nim_tool_get_audio_text_async(const char *json_audio_info, const char *json_extension, nim_tool_get_audio_text_cb_func cb, const void *user_data);

例:

	void CallbackGetAudioText(int rescode, const char *text, const char *json_extension, const void *user_data)
	{
		...
	}

	typedef void(*nim_tool_get_audio_text_async)(const char *json_audio_info, const char *json_extension, nim_tool_get_audio_text_cb_func cb, const void *user_data);
	
	void foo()
	{
		Json::Value json_value;
		json_value[nim::kNIMTransAudioKeyMime] = ; //mime类型
		json_value[nim::kNIMTransAudioKeySample] = ; //采样率
		json_value[nim::kNIMTransAudioKeyAudioUrl] = ; //下载地址
		json_value[nim::kNIMTransAudioKeyDuration] = ; //时长（毫秒）

		nim_tool_get_audio_text_async func = (nim_tool_get_audio_text_async) GetProcAddress(hInst, "nim_tool_get_audio_text_async");
		func(json_value.toStyledString().c_str(), nullptr, &CallbackGetAudioText, nullptr);
	}

## <span id="群组功能">群组功能</span>

### <span id="群组功能概述">群组功能概述</span>

云信SDK提供了普通群(`kNIMTeamTypeNormal`)，以及高级群(`kNIMTeamTypeAdvanced`)两种形式的群聊功能。高级群拥有更多的权限操作，两种群聊形式在共有操作上保持了接口一致。在群组中，当前会话的ID就是群组的ID。

- 普通群

普通群没有权限操作，适用于快速创建多人会话的场景。每个普通群只有一个管理员。管理员可以对群进行增减员操作，普通成员只能对群进行增员操作。在添加新成员的时候，并不需要经过对方同意。

- 高级群

高级群在权限上有更多的限制，权限分为群主、管理员、以及群成员。在添加成员的时候需要对方接受邀请。高级群的群成员资料提供了实时同步功能，并提供了群开关设置字段、第三方扩展字段（仅负责存储和透传）和第三方服务器扩展字段（该配置项只能通过服务器接口设置，对客户端只读）。

- 群操作权限对比


| 群操作  |普通群   |高级群   |
|---|---|---|
| 邀请成员  |群主、普通成员   |群主、管理员、普通成员   |
| 踢出成员  | 群主  |群主、管理员(管理员之间无法互相踢)   |
| 解散群  | 群主  |群主   |
| 退群  | 任何人  | 任何人  |
| 处理入群申请   |/   | 群主、管理员  |
| 更改群名称  |任何人   |群主、管理员   |
| 更改群公告  |任何人   |群主、管理员   |
| 更新验证方式  |/   |群主、管理员   |
| 添加(删除)管理员  |/   |群主   |
| 移交群主  |/   |群主   |							

###<span id="群聊消息">群聊消息</span>

群聊消息收发和管理与双人聊天完全相同，仅在消息类型(kNIMMsgKeyToType)上做了区分。

###<span id="获取群组">获取群组</span>

SDK 在程序启动时会对本地群信息进行同步，所以只需要调用本地缓存接口获取群就可以了。SDK 提供了批量获取自己的群接口、以及根据单个群 ID 查询的接口。同样SDK 也提供了远程获取群信息的接口。

- 本地获取

		void nim_team_query_all_my_teams_async(const char *json_extension, nim_team_query_all_my_teams_cb_func cb, const void *user_data);

		void nim_team_query_all_my_teams_info_async(const char *json_extension, nim_team_query_all_my_teams_info_cb_func cb, const void *user_data);

- 远程获取

		void nim_team_query_team_info_online_async(const char *tid, const char *json_extension, nim_team_opt_cb_func cb, const void *user_data);

###<span id="创建群组">创建群组</span>

云信群组分为两类：普通群和高级群，两种群组的消息功能都是相同的，区别在于管理功能。

普通群所有人都可以拉人入群，除群主外，其他人都不能踢人。

固定群则拥有完善的成员权限体系及管理功能。创建群的接口相同，传入不同的类型参数即可。

	void nim_team_create_team_async(
	const char *team_info, //群信息
	const char *jsonlist_uids, //邀请的成员
	const char *invitation_postscript, //邀请附言
	const char *json_extension, //附加数据
	nim_team_opt_cb_func cb,
	const void *user_data);

例:
	
	void CallbackCreateTeam(int error, int team_event, const char *tid, const char* str, const char *json_exten, const void *user_data)
	{
		...
	}	

	typedef void(*nim_team_create_team_async)(const char *team_info, const char *jsonlist_uids, const char *invitation_postscript, const char *json_extension, nim_team_event_cb_func cb, const void* user_data);

	void foo()
	{
		Json::Value team_info;
		team_info[kNIMTeamInfoKeyName] = ; //群名称
		team_info[kNIMTeamInfoKeyType] = ; //群类型
		team_info[kNIMTeamInfoKeyIntro] = ; //群介绍
		team_info[kNIMTeamInfoKeyJoinMode] = ; //群验证方式
		team_info[kNIMTeamInfoKeyAnnouncement] = ; //群公告

		Json::Value team_member;
		team_member.append("litianyi01");	
		team_member.append("litianyi02");
	
		nim_team_create_team_async func = (nim_team_create_team_async) GetProcAddress(hInst, "nim_team_create_team_async");
		func(team_info.toStyledString().c_str(), team_member.toStyledString().c_str(), "welcome to new team", nullptr, &CallbackCreateTeam, nullptr);
	}

###<span id="加入群组">加入群组</span>
用户可以通过被动接受邀请和主动加入两种方式进入群组。

- 邀请用户入群:

		void nim_team_invite_async(const char *tid, const char *jsonlist_uids, const char *invitation_postscript, const char *json_extension, nim_team_opt_cb_func cb, const void *user_data);

	请求完成后，如果是普通群，被邀请者将直接入群；如果是高级群，云信服务器会下发一条系统消息到目标用户，目标用户可以选择同意或者拒绝入群。

- 同意群邀请(仅限高级群):

		void nim_team_accept_invitation_async(const char *tid, const char *invitor, const char *json_extension, nim_team_opt_cb_func cb, const void *user_data);

- 拒绝群邀请(仅限高级群):

		void nim_team_reject_invitation_async(const char *tid, const char *invitor, const char *reason, const char *json_extension, nim_team_opt_cb_func cb, const void *user_data);

- 主动申请入群:

		void nim_team_apply_join_async(const char *tid, const char *reason, const char *json_extension, nim_team_opt_cb_func cb, const void *user_data);

	请求完成后，如果是普通群，将直接入群；如果是高级群，云信服务器会下发一条系统消息给群管理员，管理员可以选择通过或者拒绝申请。

- 通过申请(仅限高级群):

		void nim_team_pass_join_apply_async(const char *tid, const char *applicant_id, const char *json_extension, nim_team_opt_cb_func cb, const void *user_data);

- 拒绝申请(仅限高级群):

		void nim_team_reject_join_apply_async(const char *tid, const char *applicant_id, const char *reason, const char *json_extension, nim_team_opt_cb_func cb, const void *user_data);
	
###<span id="拉人入群">拉人入群</span>
普通群所有人都可以拉人入群，高级群仅管理员和拥有者可以邀请人入群，接口均为：

	void nim_team_invite_async(const char *tid
							, const char *jsonlist_uids
							, const char *invitation_postscript
							, const char *json_extension
							, nim_team_opt_cb_func cb
							, const void *user_data);

例：

	void CallbackTeamInvite(int error, int team_event, const char *tid, const char* str, const char *json_exten, const void *user_data)
	{
		...
	}

	typedef void(*nim_team_invite_async)(const char *tid, const char *jsonlist_uids, const char *invitation_postscript, const char *json_extension, nim_team_event_cb_func cb, const void* user_data);

	void foo()
	{
		Json::Value uids_;
		uids_.append("pc1");
		uids_.append("pc2");
		nim_team_invite_async func = (nim_team_invite_async) GetProcAddress(hInst, "nim_team_invite_async");
		func("tid", uids_.toStyledString().c_str(), "", nullptr, &CallbackTeamInvite, nullptr);
	}


###<span id="踢人出群">踢人出群</span>
普通群仅拥有者可以踢人，高级群拥有者和管理员可以踢人，且管理员不能踢拥有者和其他管理员。

	void nim_team_kick_async(const char *tid
							, const char *jsonlist_uids
							, const char *json_extension
							, nim_team_opt_cb_func cb
							, const void *user_data);

###<span id="主动退群">主动退群</span>
除拥有者外，其他用户均可以主动退群：

	void nim_team_leave_async(const char *tid
							, const char *json_extension
							, nim_team_opt_cb_func cb
							, const void *user_data);

###<span id="编辑群组资料">编辑群组资料</span>

普通群所有人均可以修改群名，高级群仅拥有者和管理员可修改群名及其他群资料。

	void nim_team_update_team_info_async(const char *tid, const char *json_info, const char *json_extension, nim_team_opt_cb_func cb, const void *user_data);

例:

	void CallbackTeamOperate(int error, int team_operate, const char *tid, const char* str, const char *json_exten, const void *user_data)
	{
		if (error == kNIMResSuccess)
		{
			...
		}
		else
		{
			...
		}
	}

	typedef void(*nim_team_update_team_info_async)(const char *tid, const char *json_info, const char *json_extension, nim_team_event_cb_func cb_func, const void* user_data);

	void foo()
	{
		Json::Value values;
		values[kNIMTeamInfoKeyID] = "tid";	
		values[kNIMTeamInfoKeyAnnouncement] = "123"; //修改群公告，同样的，目前可以修改群名称，群简介，具体参阅api文档
		values[kNIMTeamInfoKeyBits] = 1; //修改群开关设置（如开启或关闭群消息提醒），具体参阅api文档里NIMTeamBitsConfigMask（群组信息Bits属性kNIMTeamInfoKeyBits的配置定义）

		nim_team_update_team_info_async func = (nim_team_update_team_info_async) GetProcAddress(hInst, "nim_team_update_team_info_async");
		func("tid", values.toStyledString().c_str(), nullptr, &CallbackTeamOperate, nullptr);
	}
	
###<span id="管理群组权限">管理群组权限</span>

高级群群主可以对群进行权限管理，权限管理包括: 

- 提升管理员(仅限高级群):

		void nim_team_add_managers_async(const char *tid //群组id
									, const char *jsonlist_admin_ids //提升管理员ids
									, const char *json_extension
									, nim_team_opt_cb_func cb
									, const void *user_data);

- 移除管理员(仅限高级群):

		void nim_team_remove_managers_async(const char *tid //群组id
									, const char *jsonlist_admin_ids //移除管理员ids
									, const char *json_extension
									, nim_team_opt_cb_func cb
									, const void *user_data);

- 转让群(仅限高级群):

		void nim_team_transfer_team_async(const char *tid //群组id
									, const char *new_owner_id //移交对象uid
									, bool is_leave //是否同时退出群
									, const char *json_extension
									, nim_team_opt_cb_func cb
									, const void *user_data);

###<span id="群组成员">群组成员</span>

- 获取群成员列表，获取到的群成员只有云信服务器托管的群相关数据，需要开发者结合自己管理的用户数据进行界面显示。
	
		void nim_team_query_team_members_async(const char *tid, bool include_user_info, const char *json_extension, nim_team_query_team_members_cb_func cb, const void *user_data);

	例:

		void CallbackQueryTeamMembersCb(const char * tid, int count, bool include_user_info, const char* str, const char *json_exten, const void *user_data)
		{
			//解析str
		}

		typedef void(*nim_team_query_team_members_async)(const char *tid, bool include_user_info, const char *json_extension, nim_team_query_team_members_cb_func cb, const void* user_data);

		void foo()
		{
			nim_team_query_team_members_async func = (nim_team_query_team_members_async) GetProcAddress(hInst, "nim_team_query_team_members_async");
			func("tid", include_user_info ? true : false, nullptr, &CallbackQueryTeamMembersCb, nullptr);
		}

- 用户退群
	
		void nim_team_leave_async(const char *tid, const char *json_extension, nim_team_opt_cb_func cb, const void *user_data);

- 踢出用户

		void nim_team_kick_async(const char *tid, const char *jsonlist_uids, const char *json_extension, nim_team_opt_cb_func cb, const void *user_data);

- 解散群

		void nim_team_dismiss_async(const char *tid, const char *json_extension, nim_team_opt_cb_func cb, const void *user_data);

###<span id="群组通知">群组通知</span>

用户在创建群或者进入群成功之后，任何关于群的变动（群组资料变动，群成员变动等），云信服务器都会下发一条群通知消息。APP 可以通过注册全局回调函数接受群组通知。

	void nim_team_reg_team_event_cb(const char *json_extension, nim_team_event_cb_func cb, const void *user_data);

例:

	void CallbackTeamEvent(int error, int team_event, const char *tid, const char* str, const char *json_exten, const void *user_data)
	{
		switch (team_event)
		{
		case kNIMNotificationIdLocalCreateTeam:
			...
		...	
		}
	}
	
	typedef void(*nim_team_reg_team_event_cb)(const char *json_extension, nim_team_event_cb_func cb, const void *user_data);

	void foo()
	{
		nim_team_reg_team_event_cb func = (nim_team_reg_team_event_cb) GetProcAddress(hInst, "nim_team_reg_team_event_cb");
		func(nullptr, &CallbackTeamEvent, nullptr);
	}

- SDK 在收到群通知之后，会对本地缓存的群信息做出对应的修改，然后触发与修改相对应的委托事件回调。
- 群通知是接收型的消息，开发者不应该自己手动去创建和发送群通知消息。

###<span id="自定义拓展">自定义拓展</span>

SDK 提供了群信息的拓展接口，开发者可以通过维护群信息的两个属性来自行定义内容。

- 应用方可以自行拓展这个字段做个性化配置，客户端不可以修改这个字段 kNIMTeamInfoKeyServerCustom (nim\_team\_def\.h)
	
- 应用方可以自行拓展这个字段做个性化配置，客户端可以修改这个字段 kNIMTeamInfoKeyCustom (nim\_team\_def\.h)

##<span id="历史记录">历史记录</span>
###<span id="本地记录">本地记录</span>
- 获取聊天对象的本地消息记录

		void nim_msglog_query_msg_async(const char *account_id //会话id，对方的account id或者群组tid
								, NIMSessionType to_type //会话类型
								, int limit_count //一次查询数量，建议20
								, __int64 last_time //上次查询最后一条消息的时间戳（按时间逆序起查，即最小的时间戳）
								, const char *json_extension
								, nim_msglog_query_cb_func cb
								, const void *user_data);

	例：
	
		//按时间逆序查询，逆序排列
		void CallbackQueryMsgCb(int res_code, const char* id, const char* type, const char* str, const char *json_exten, const void *user_data)
		{
			Json::Value values;
			Json::Reader reader;
			if (reader.parse(str, values) && values.isObject())
			{
				int count = values[kNIMMsglogQueryKeyCount].asInt();
				if (count > 0)
					...
			}
		}

		typedef void(*nim_msglog_query_msg_async)(const char* account_id, nim::NIMSessionType to_type, int limit_count, __int64 last_time, const char *json_extension, nim_msglog_query_cb_func cb, const void* user_data);

		void foo()
		{
			nim_msglog_query_msg_async func = (nim_msglog_query_msg_async) GetProcAddress(hInst, "nim_msglog_query_msg_async");
			func("acount_id", is_team ? kNIMSessionTypeTeam : kNIMSessionTypeP2P, 20, 0, nullptr, &CallbackQueryMsgCb, nullptr);
		}

- 删除消息记录：

		//删除单条消息
		void nim_msglog_delete_async(const char *account_id, NIMSessionType to_type, const char *msg_id, const char *json_extension, nim_msglog_res_cb_func cb, const void *user_data);

		//删除与某个聊天对象的全部消息记录
		void nim_msglog_batch_status_delete_async(const char *account_id, NIMSessionType to_type, const char *json_extension, nim_msglog_res_ex_cb_func cb, const void *user_data);

		//删除指定会话类型的所有消息
		void nim_msglog_delete_by_session_type_async(bool delete_sessions, NIMSessionType to_type, const char *json_extension, nim_msglog_res_ex_cb_func cb, const void *user_data);

		//删除所有消息记录
		void nim_msglog_delete_all_async(bool delete_sessions, const char *json_extension, nim_msglog_res_cb_func cb, const void *user_data);

nim\_msglog.h 文件中的接口提供了比较完善的消息记录管理功能，开发者可本地查阅、在线查阅、删除和清空聊天记录，还可以调用nim\_msglog\_write\_db\_only\_async 接口只往本地消息历史数据库里写入一条消息（通常是APP 的本地自定义消息，并不会发给服务器）。详见API 文档。此外，还要注意：本地消息历史操作（如删除）及状态变化（如发送失败或成功）会与会话列表里的消息状态自动同步，App 上层需要根据nim\_session\_reg\_change\_cb 注册好的会话列表通知回调进行相应通知事件的处理。

###<span id="导入导出">导入导出</span>
- 导入消息历史记录（不包括系统通知，且不允许拿别人的消息历史记录文件来导入）:

		void nim_msglog_import_db_async(const char *src_path, const char *json_extension, nim_msglog_res_cb_func res_cb, const void *res_user_data, nim_msglog_import_prg_cb_func prg_cb, const void *prg_user_data);

	例:

		void CallbackMsglogRes(int res, const char* msg_id, const char *json_exten, const void *user_data)
		{
			...
		}

		void CallbackMsgImportPrg(int res_code, const char* id, const char* type, const char* str, const char *json_exten, const void *user_data)
		{
			...
		}

		typedef void(*nim_msglog_import_db_async)(const char *src_path, const char *json_extension, nim_msglog_res_cb_func res_cb, const void *res_user_data, nim_msglog_import_prg_cb_func prg_cb, const void *prg_user_data);

		void foo()
		{
			nim_msglog_import_db_async func = (nim_msglog_import_db_async) GetProcAddress(hInst, "nim_msglog_import_db_async");
			func("src_path", "", &CallbackMsglogRes, nullptr, &CallbackMsgImportPrg, nullptr);
		}

- 导出整个消息历史DB文件（不包括系统通知）：
	
		void nim_msglog_export_db_async(const char *dst_path, const char *json_extension, nim_msglog_res_cb_func cb, const void *user_data);

###<span id="云端记录">云端记录</span>

- 在线查询聊天对象的消息历史记录：

		void nim_msglog_query_msg_online_async(const char *id //会话id，对方的account id或者群组tid
										, NIMSessionType to_type //会话类型
										, int limit_count //本次查询的消息条数上限(最多100条)
										, __int64 from_time //起始时间点，单位：毫秒
										, __int64 end_time //结束时间点，单位：毫秒
										, __int64 end_msg_id //结束查询的最后一条消息的server_msg_id(不包含在查询结果中) 
										, bool reverse //true：反向查询(按时间正序起查，正序排列)，false：按时间逆序起查，逆序排列（建议默认为false）
										, bool need_save_to_local //true: 将在线查询结果保存到本地，false: 不保存
										, const char *json_extension
										, nim_msglog_query_cb_func cb
										, const void *user_data);

	例:
	
		void CallbackQueryMsgCb(int res_code, const char* id, const char* type, const char* str, const char *json_exten, const void *user_data)
		{
			...
		}

		typedef void(*nim_msglog_query_msg_online_async)(const char *id, nim::NIMSessionType to_type, int limit_count, __int64 from_time, __int64 end_time, __int64 end_msg_id, bool reverse,	bool need_save_to_local, const char *json_extension, nim_msglog_query_cb_func cb, const void *user_data);

		void foo()
		{
			nim_msglog_query_msg_online_async func = (nim_msglog_query_msg_online_async) GetProcAddress(hInst, "nim_msglog_query_msg_online_async");
			func("acount_id", is_team ? kNIMSessionTypeTeam : kNIMSessionTypeP2P, 20, 0, msg_time, first_msg_server_msg_id, false, false, nullptr, &CallbackQueryMsgCb, nullptr);
		}
			
## <span id="音视频网络通话">音视频网络通话</span>

###<span id="初始化与清理">初始化与清理</span>

在调用音视频网络通话相关功能前，需要在nim.dll所在目录中有对应版本的nim\_videochat.dll文件。

    初始化NIM VCHAT，需要在SDK的nim_client_init成功之后
    bool nim_vchat_init(const char *json_extension);
    	param[in] json_extension 无效的扩展字段
    	return bool 初始化结果，如果是false则音视频网络通话所有接口调用无效

    清理NIM VCHAT，需要在SDK的nim_client_cleanup之前
    void nim_vchat_cleanup(const char *json_extension);
    	param[in] json_extension 无效的扩展字段
    	return void 无返回值

###<span id="设置通话回调函数">设置通话回调函数</span>

在使用音视频网络通话前，需要先设置通话回调
    
    注册通话回调函数
    void nim_vchat_set_cb_func(nim_vchat_cb_func cb);
    
    回调原型
    void nim_vchat_cb_func(NIMVideoChatSessionType type, __int64 channel_id, int code, const char *json_extension, const void *user_data);
    	具体参数说明见nim_vchat_def.h；在截下来的功能中也将一一说明

###<span id="发起通话">发起通话</span>

发起一个音视频网络通话，结果将在nim\_vchat\_cb\_func中返回
    
    启动通话
    bool nim_vchat_start(NIMVideoChatMode mode, const char* json_extension, const void *user_data)
    	param[in] mode NIMVideoChatMode 启动音视频通话类型 见nim_vchat_def.h
    	param[in] json_extension Json string 扩展，kNIMVChatUids成员id列表 如{"uids":["uid_temp"]}
    	param[in] user_data 无效的扩展字段
    	return bool true 调用成功，false 调用失败可能有正在进行的通话

    异步回调见nim_vchat_cb_func中type为以下值时
    	kNIMVideoChatSessionTypeStartRes,			//创建通话结果 code=200成功，json无效
		kNIMVideoChatSessionTypeConnect,			//通话中链接状态通知 code对应NIMVChatConnectErrorCode， 非200均失败并底层结束 json无效

###<span id="收到通话邀请">收到通话邀请</span>

通话邀请由nim\_vchat\_cb\_func通知，type值对应kNIMVideoChatSessionTypeInviteNotify，其中code无效，json返回 kNIMVChatUid为发起者，kNIMVChatType对应NIMVideoChatMode

###<span id="通话邀请回应">通话邀请回应</span>

通知邀请方自己的处理结果，结果将在nim\_vchat\_cb\_func中返回
    
	邀请回应接口
	bool nim_vchat_callee_ack(__int64 channel_id, bool accept, const char* json_extension, const void *user_data)
    	param[in] channel_id 音视频通话通道id
    	param[in] accept true 接受，false 拒绝
		param[in] json_extension 无效的扩展字段
    	param[in] user_data 无效的扩展字段
		return bool true 调用成功，false 调用失败（可能channel_id无匹配，如要接起另一路通话前先结束当前通话）

    异步回调见nim_vchat_cb_func中type为以下值时
    	kNIMVideoChatSessionTypeCalleeAckRes,		//接受拒绝结果 json 无效 code: 200:成功 9103 : 已经在其他端接听或拒绝过这通电话

###<span id="收到被邀请方的回应">收到被邀请方的回应</span>

被邀请方的回应由nim\_vchat\_cb\_func通知，type值对应kNIMVideoChatSessionTypeCalleeAckNotify，其中code无效，json返回 kNIMVChatUid为发起者，kNIMVChatType对应VideoChatMode, kNIMVChatAccept

###<span id="收到自己在其他端的同步回应通知">收到自己在其他端的同步回应通知</span>

本人在其他端回应了由nim\_vchat\_cb\_func通知，type值对应kNIMVideoChatSessionTypeSyncAckNotify，其中code无效，json返回 kNIMVChatTime，kNIMVChatType（对应NIMVideoChatMode），kNIMVChatAccept，kNIMVChatClient

###<span id="设置通话类型">设置通话类型</span>

开始通话，或者和对方协商转换通话类型后，告诉SDK当前的通话类型（不告知对方）

    bool nim_vchat_set_talking_mode(NIMVideoChatMode mode, const char* json_extension)
    	param[in] mode NIMVideoChatMode 音视频通话类型 见nim_vchat_def.h
    	param[in] json_extension 无效的扩展字段
    	return bool true 调用成功，false 调用失败

###<span id="通话控制">通话控制</span>

协商或通知对方当前会话的一些变化

    bool nim_vchat_control(__int64 channel_id, NIMVChatControlType type, const char* json_extension, const void *user_data)
    	param[in] channel_id 音视频通话通道id
    	param[in] type NIMVChatControlType 见nim_vchat_def.h
    	param[in] json_extension 无效的扩展字段
    	param[in] user_data 无效的扩展字段
    	return bool true 调用成功，false 调用失败

    异步回调见nim_vchat_cb_func中type为以下值时
    	kNIMVideoChatSessionTypeControlRes,		//code=200成功，json返回 kNIMVChatType对应NIMVChatControlType

###<span id="对方通话控制的通知">对方通话控制的通知</span>

收到对方的通话控制消息，由nim\_vchat\_cb\_func通知，type值对应kNIMVideoChatSessionTypeControlNotify，其中code无效，json返回 kNIMVChatUid为发起者，kNIMVChatType对应NIMVChatControlType, kNIMVChatAccept

###<span id="通话中成员进出">通话中成员进出</span>

通话中成员进出由nim\_vchat\_cb\_func通知，type值对应kNIMVideoChatSessionTypePeopleStatus，其中code对应NIMVideoChatSessionStatus, json返回kNIMVChatUid，对方离开后应主动结束通话

###<span id="通话中网络变化">通话中网络变化</span>

通话中网络变化由nim\_vchat\_cb\_func通知，type值对应kNIMVideoChatSessionTypeNetStatus，其中code对应NIMVideoChatSessionNetStat, json无效

###<span id="结束通话">结束通话</span>

需要在通话结束后调用，用于底层挂断和清理数据，如果是自己主动结束会通知对方

    void nim_vchat_end(const char* json_extension)
    	NIM VCHAT 结束通话(需要主动在通话结束后调用，用于底层挂断和清理数据)
    	param[in] json_extension 无效的扩展字段
    	return void 无返回值

    如果是自己主动结束会通知对方，异步回调见nim_vchat_cb_func中type为以下值时
    	kNIMVideoChatSessionTypeHangupRes,		//code=200成功，json无效

###<span id="收到对方结束通知">收到对方结束通知</span>

收到对方结束由nim\_vchat\_cb\_func通知，type值对应kNIMVideoChatSessionTypeHangupNotify，其中code无效，json无效

## <span id="音视频设备">音视频设备</span>
提供麦克风、扬声器（听筒）、摄像头等设备的遍历、启动关闭、监听的接口，使用相关功能前需要调用OleInitialize来初始化COM库

###<span id="遍历设备">遍历设备</span>

在启动设备前或者设备有变更后需要遍历设备，其中kNIMDeviceTypeAudioOut和kNIMDeviceTypeAudioOutChat都为听筒设备，遍历的时候等同

    void nim_vchat_enum_device_devpath(NIMDeviceType type, const char *json_extension, nim_vchat_enum_device_devpath_sync_cb_func cb, const void *user_data) 
    	param[in] type NIMDeviceType 见nim_device_def.h
    	param[in] json_extension 无效的扩展字段
    	param[in] cb 结果回调见nim_device_def.h
    	param[in] user_data 无效的扩展字段
    	return void 无返回值，设备信息由同步返回的cb通知

###<span id="启动设备">启动设备</span>

可以启动麦克风、摄像头、本地采集音频播放器、对方语音播放器，设备路径填空的时候会启动系统首选项

    启动设备，同一NIMDeviceType下设备将不重复启动，不同的设备会先关闭前一个设备开启新设备
    void nim_vchat_start_device(NIMDeviceType type, const char* device_path, unsigned fps, const char *json_extension, nim_vchat_start_device_cb_func cb, const void *user_data)
    	param[in] type NIMDeviceType 见nim_device_def.h
		param[in] device_path 设备路径为nim_vchat_enum_device_devpath_sync_cb_func结果中的kNIMDevicePath
		param[in] fps 摄像头为采样频率（一般传电源频率取50）,其他NIMDeviceType无效（麦克风采样频率由底层控制，播放器采样频率也由底层控制）
		param[in] json_extension 无效的扩展字段
		param[in] cb 结果回调见nim_device_def.h
		param[in] user_data 无效的扩展字段
		return void 无返回值

###<span id="结束设备">结束设备</span>
    
    void nim_vchat_end_device(NIMDeviceType type, const char *json_extension)
    	param[in] type NIMDeviceType 见nim_device_def.h
    	param[in] json_extension 无效的扩展字段
    	return void 无返回值

###<span id="监听设备状态">监听设备状态</span>

通过设置设备监听，可以定时获知设备状态并启动非主动结束的设备，这里只对麦克风和摄像头有效
    
    void nim_vchat_add_device_status_cb(NIMDeviceType type, nim_vchat_device_status_cb_func cb)
    	param[in] type NIMDeviceType（kNIMDeviceTypeAudioIn和kNIMDeviceTypeVideo有效） 见nim_device_def.h
    	param[in] cb 结果回调见nim_device_def.h
    	return void 无返回值

由于开启设备监听后会定时检查设备状态，在不需要时结束监听

    void nim_vchat_remove_device_status_cb(NIMDeviceType type)
    	param[in] type DeviceType（kNIMDeviceTypeAudioIn和kNIMDeviceTypeVideo有效） 见nim_device_def.h
    	return void 无返回值

###<span id="监听音频数据">监听音频数据</span>

通过设置监听，可以获得音频的pcm数据包，通过启动设备kDeviceTypeAudioOut和kDeviceTypeAudioOutChat由底层播放，可以不监听
    
    void nim_vchat_set_audio_data_cb(bool capture, nim_vchat_audio_data_cb_func cb)
    	param[in] capture true 标识监听麦克风采集数据，false 标识监听通话中对方音频数据
    	param[in] cb 结果回调见nim_device_def.h
    	return void 无返回值

###<span id="监听视频数据">监听视频数据</span>

通过设置监听，可以获得视频的ARGB数据包

    void nim_vchat_set_video_data_cb(bool capture, nim_vchat_video_data_cb_func cb)
    	param[in] capture true 标识监听采集数据，false 标识监听通话中对方视频数据
    	param[in] cb 结果回调见nim_device_def.h
    	return void 无返回值

###<span id="音频设置">音频设置</span>

设置音频音量，默认255,且音量均由软件换算得出,设置麦克风音量自动调节后麦克风音量参数无效

    void nim_vchat_set_audio_volumn(unsigned char volumn, bool capture)
    	param[in] volumn 结果回调见nim_device_def.h
    	param[in] capture true 标识设置麦克风音量，false 标识设置播放音量
    	return void 无返回值

获取音频音量

    unsigned char nim_vchat_get_audio_volumn(bool capture)
    	param[in] capture true 标识获取麦克风音量，false 标识获取播放音量
    	return unsigned char 音量值

设置麦克风音量自动调节
    
    void nim_vchat_set_audio_input_auto_volumn(bool auto_volumn)
    	param[in] auto_volumn true 标识麦克风音量自动调节，false 标识麦克风音量不调节，这时nim_vchat_set_audio_volumn中麦克风音量参数起效
    	return void 无返回值

获取是否自动调节麦克风音量

    bool nim_vchat_get_audio_input_auto_volumn()
    	return bool true 标识麦克风音量自动调节，false 标识麦克风音量不调节

###<span id="音视频通话示例">音视频通话示例</span>
对所有异步回调接口，在实现时post到ui线程处理

	//本地设备遍历
	struct MEDIA_DEVICE_DRIVE_INFO
	{
		std::string device_path_;
		std::string friendly_name_;
	};
	std::list<MEDIA_DEVICE_DRIVE_INFO> list_audioin_;
	std::list<MEDIA_DEVICE_DRIVE_INFO> list_audioout_;
	std::list<MEDIA_DEVICE_DRIVE_INFO> list_video_;
	void GetDeviceByJson(std::list<MEDIA_DEVICE_DRIVE_INFO>& list, const char* json)
	{
		Json::Value valus;
		Json::Reader reader;
		if (reader.parse(json, valus) && valus.isArray())
		{
			int num = valus.size();
			for (int i=0;i<num;++i)
			{
				Json::Value device;
				device = valus.get(i, device);
				MEDIA_DEVICE_DRIVE_INFO info;
				info.device_path_ = device[kNIMDevicePath].asString();
				info.friendly_name_ = device[kNIMDeviceName].asString();
				list.push_back(info);
			}
		}
	}
	//遍历设备的回调为同步接口，不需要post
	void EnumDevCb(bool ret, NIMDeviceType type, const char* json, const void*)
	{
		Print("EnumDevCb: ret=%d, NIMDeviceType=%d\r\njson: %s", ret, type, json);
		if (ret)
		{
			switch (type)
			{
			case kNIMDeviceTypeAudioIn:
				GetDeviceByJson(list_audioin_, json);
				break;
			case kNIMDeviceTypeAudioOut:
				GetDeviceByJson(list_audioout_, json);
				break;
			case kNIMDeviceTypeVideo:
				GetDeviceByJson(list_video_, json);
				break;
			}
		}
	}
	void EnumDev()
	{
		nim_vchat_enum_device_devpath(kNIMDeviceTypeVideo, "", EnumDevCb, nullptr);
		nim_vchat_enum_device_devpath(kNIMDeviceTypeAudioIn, "", EnumDevCb, nullptr);
		nim_vchat_enum_device_devpath(kNIMDeviceTypeAudioOut, "", EnumDevCb, nullptr);
	}

	//监听设备回调
	void AudioStatus(NIMDeviceType type, UINT status, const char* path, const char*)
	{
		//
	}
	void VideoStatus(NIMDeviceType type, UINT status, const char* path, const char*)
	{
		//
	}

	//监听设备,建议在需要的时候开启，不需要的时候调用停止监听
	void AddDeviceStatusCb()
	{
		nim_vchat_add_device_status_cb(kNIMDeviceTypeVideo, VideoStatus);
		nim_vchat_add_device_status_cb(kNIMDeviceTypeAudioIn, AudioStatus);
	}
	//停止监听设备
	void RemoveDeviceStatusCb()
	{
		nim_vchat_remove_device_status_cb(kNIMDeviceTypeVideo);
		nim_vchat_remove_device_status_cb(kNIMDeviceTypeAudioIn);
	}

	//音视频通话
	//通话回调函数
	void OnVChatEvent(NIMVideoChatSessionType type, __int64 channel_id, int code, const char *json_extension, const void *user_data)
	{
		std::string json = json_extension;
		switch (type)
		{
		case nim::kNIMVideoChatSessionTypeStartRes:{
			if (code == 200)
			{
				//发起通话成功
			}
			else
			{
				EndVChat();			
			}
		}break;
		case nim::kNIMVideoChatSessionTypeInviteNotify:{
			Json::Value valus;
			Json::Reader reader;
			if (reader.parse(json, valus))
			{
				std::string uid = valus[nim::kNIMVChatUid].asString();
				int mode = valus[nim::kNIMVChatType].asInt();
				bool exist = false;//是否已经有音视频通话存在
				if (exist)
				{
					//通知对方忙碌
					VChatControl(channel_id, nim::kNIMTagControlBusyLine);
					//拒绝接听，可由用户在ui触发
					VChatCalleeAck(channel_id, false);
				}
				else
				{
					//通知对方收到邀请
					VChatControl(channel_id, nim::kNIMTagControlReceiveStartNotifyFeedback);
					//接听，可由用户在ui触发
					VChatCalleeAck(channel_id, true);
				}
			}
		}break;
		case nim::kNIMVideoChatSessionTypeCalleeAckRes:{
			Json::Value valus;
			Json::Reader reader;
			if (reader.parse(json, valus))
			{
				//std::string uid = valus[nim::kNIMVChatUid].asString();
				bool accept = valus[nim::kNIMVChatAccept].asInt() > 0;
				if (accept && code != 200)
				{
					//接听失败，结束当前通话
					EndVChat();	
				}
			}
		}break;
		case nim::kNIMVideoChatSessionTypeCalleeAckNotify:{
			Json::Value valus;
			Json::Reader reader;
			if (reader.parse(json, valus))
			{
				//std::string uid = valus[nim::kNIMVChatUid].asString();
				bool accept = valus[nim::kNIMVChatAccept].asInt() > 0;
				if (!accept)
				{
					//对方拒绝，结束当前通话
					EndVChat();	
				}
			}
		}break;
		case nim::kNIMVideoChatSessionTypeControlRes:{
			//控制命令发送结果
		}break;
		case nim::kNIMVideoChatSessionTypeControlNotify:{
			Json::Value valus;
			Json::Reader reader;
			if (reader.parse(json, valus))
			{
				//std::string uid = valus[nim::kNIMVChatUid].asString();
				int type = valus[nim::kNIMVChatType].asInt();
				//收到一个控制通知
			}
		}break;
		case nim::kNIMVideoChatSessionTypeConnect:{
			if (code != 200)
			{
				//连接音视频服务器失败
			}
			else
			{
				//连接音视频服务器成功，启动麦克风、听筒、摄像头
				StartDevice();
			}
		}break;
		case nim::kNIMVideoChatSessionTypePeopleStatus:{
			if (code == nim::kNIMVideoChatSessionStatusJoined)
			{
				//对方进入
			}
			else if (code == nim::kNIMVideoChatSessionStatusLeaved)
			{
				//对方离开，结束
				EndVChat();	
			}
		}break;
		case nim::kNIMVideoChatSessionTypeNetStatus:{
			//网络状况变化
		}break;
		case nim::kNIMVideoChatSessionTypeHangupRes:{
			//挂断结果
		}break;
		case nim::kNIMVideoChatSessionTypeHangupNotify:{
			//收到对方挂断
		}break;
		case nim::kNIMVideoChatSessionTypeSyncAckNotify:{
			Json::Value valus;
			Json::Reader reader;
			if (reader.parse(json, valus))
			{
				bool accept = valus[nim::kNIMVChatAccept].asInt() > 0;
				//在其他端处理
			}
		}break;
		}
	}
	//一个简单的视频数据绘制
	void DrawPic(bool left, unsigned int width, unsigned int height, const char* data)
	{
		if (!IsWindow(m_hWnd))
		{
			return;
		}
		int pos = 10;
		const char* scr_data = data;
		int w = 0;
		int top = 30;
		if (!left)
		{
			RECT rect;
			GetWindowRect(m_hWnd, &rect);
			w = rect.right - rect.left;
			pos = w - width - 20;
		}
		// 构造位图信息头
		BITMAPINFOHEADER bih;
		memset(&bih, 0, sizeof(BITMAPINFOHEADER));
		bih.biSize = sizeof(BITMAPINFOHEADER);
		bih.biPlanes = 1;
		bih.biBitCount = 32;
		bih.biWidth = width;
		bih.biHeight = height;
		bih.biCompression = BI_RGB;
		bih.biSizeImage = width * height * 4;
	
		// 开始绘制
		HDC	hdc = ::GetDC(m_hWnd);
		PAINTSTRUCT ps;
		::BeginPaint(m_hWnd, &ps);
		// 使用DIB位图和颜色数据对与目标设备环境相关的设备上的指定矩形中的像素进行设置
		SetDIBitsToDevice(
			hdc, pos, top,
			bih.biWidth, bih.biHeight,
			0, 0, 0, bih.biWidth,
			scr_data,
			(BITMAPINFO*)&bih,
			DIB_RGB_COLORS);
		::EndPaint(m_hWnd, &ps);
		::ReleaseDC(m_hWnd, hdc);
	}
	//本地视频数据监听
	void VideoCaptureData(unsigned __int64 time, const char* data, unsigned int size, unsigned int width, unsigned int height, const char*)
	{
		static int capture_video_num = 0;
		capture_video_num++;
		if (capture_video_num % 2 == 0)
		{
			return;
		}
		DrawPic(true, width, height, data);
	}
	//接收视频数据监听
	void VideoRecData(unsigned __int64 time, const char* data, unsigned int size, unsigned int width, unsigned int height, const char*)
	{
		DrawPic(false, width, height, data);
	}
	//初始化回调接口
	void InitVChatCb()
	{
		nim_vchat_set_cb_func(OnVChatEvent);
		nim_vchat_set_video_data_cb(true, VideoCaptureData);
		nim_vchat_set_video_data_cb(false, VideoRecData);
	}
	//启动设备
	void StartDevice()
	{
		nim_vchat_start_device(kNIMDeviceTypeAudioIn, "", 0, "", nullptr, nullptr);
		nim_vchat_start_device(kNIMDeviceTypeAudioOutChat, "", 0, "", nullptr, nullptr);
		nim_vchat_start_device(kNIMDeviceTypeVideo, "", 50, "", nullptr, nullptr);
	}
	//停用设备
	void EndDevice()
	{
		nim_vchat_end_device(kNIMDeviceTypeVideo, "");
		nim_vchat_end_device(kNIMDeviceTypeAudioIn, "");
		nim_vchat_end_device(kNIMDeviceTypeAudioOutChat, "");
	}
	//发起邀请
	void StartVChat()
	{
		Json::FastWriter fs;
		Json::Value value;
		value[kNIMVChatUids].append("uid");
		std::string json_value = fs.write(value);
		nim_vchat_start(kNIMVideoChatModeVideo, json_value.c_str(), nullptr);
	}
	//结束通话
	void EndVChat()
	{
		EndDevice();
		//当结束当前音视频会话时都可以调用nim_vchat_end，sdk底层会对通话进行判断清理
		nim_vchat_end("");
	}

## <span id="实时会话(白板)">实时会话(白板)</span>   

###<span id="说明">说明</span>
实时会话提供了一个音视频和一个tcp的通道，允许用户同时发起这2个通道，实现数据和音视频的会话功能，详见nim\_rts.h和nim\_rts\_def.h。
其中需要注意的是，如果用户需要使用音视频通道，请使用音视频网络通话中的初始化和清理接口（nim\_vchat.h），并在会话中使用nim_device.h中的设备相关接口，设置启动设备。并且，音视频通话具有唯一性，和音视频网络通话互斥，一个客户端只允许同时存在一个音视频连接。

###<span id="发起会话">发起会话</span>
发起者通过nim\_rts\_start将创建一个rts会话。其中可以通过传入json\_extension，来修改通话的设置，参数使用见nim\_rts\_def.h。发起会话的结果将由nim\_rts\_start\_cb\_func，如果成功将得到一个唯一的session_id来标识此次会话。

###<span id="收到会话邀请">收到会话邀请</span>
被邀请者，通过nim\_rts\_set\_start\_notify\_cb\_func接口来设置回调实现，由nim\_rts\_start\_notify\_cb\_func通知。

###<span id="回复收到的邀请">回复收到的邀请</span>
被邀请者回复邀请接口为nim\_rts\_ack，可以通过传入json\_extension，来修改通话的设置，参数使用见nim\_rts\_def.h。

###<span id="发起者接收对方的响应">发起者接收对方的响应</span>
发起者通过设置nim\_rts\_set\_ack\_notify\_cb\_func接口，从回调函数nim\_rts\_ack\_notify\_cb\_func得到被邀请者的回复。

###<span id="被邀请者的多端同步">被邀请者的多端同步</span>
被邀请通过设置nim\_rts\_set\_sync\_ack\_notify\_cb\_func接口，可以知道自己其他端在收到邀请后的回复，同时sdk将会清理本次邀请。

###<span id="监听通道连接状态">监听通道连接状态</span>
当被邀请者同意邀请后，会话双方将连接服务器进入数据通道。监听通道状态，通过nim\_rts\_set\_connect\_notify\_cb\_func接口实现。如果会话中有多个通道，则各个通道的状态将分别通过nim\_rts\_connect\_notify\_cb\_func通知。其中只有返回回调中code=200时，标识通道连接成功，其他均为失败，并且sdk会清理失败的通道，但不会主动结束会话。同时如果有音视频通道，则可以在通道连接成功的时候，启动相关音视频设备。

###<span id="监听通道内成员变化">监听通道内成员变化</span>
在通道连接成功后，通过由nim\_rts\_set\_member\_change\_cb\_func设置提供的nim\_rts\_member\_change\_cb\_func回调，将通知通道内其他成员的状态。当通道内有其他成员时，数据通道才允许发送数据，否则发送数据将失败。

###<span id="会话控制通知">会话控制通知</span>
通过nim\_rts\_control接口，用户可以在会话过程中，向对方发送一些自定义的通知。接收通知由nim\_rts\_set\_control\_notify\_cb\_func接口中的nim\_rts\_control\_notify\_cb\_func回调返回。

###<span id="音视频通话模式切换">音视频通话模式切换</span>
用户可以通过nim\_rts\_set\_vchat\_mode接口切换音频或视频模式。模式切换的作用是，在非视频模式时用户的视频数据将不会发送给对方。注意，音频数据如果在设备打开的时候会通过音视频通道发送给对方，如果用户需要静音，则需要关闭麦克风设备，或者在发起会话时使用custom\_audio模式，自主控制音频数据。

###<span id="主动结束会话">主动结束会话</span>
通过nim\_rts\_hangup接口，用户可以主动结束会话。并通过nim\_rts\_set\_hangup\_notify\_cb\_func接口的nim\_rts\_hangup\_notify\_cb\_func回调，监听对方的结束通知。注意，对于一个成功发起的会话，需要主动hangup结束或者收到对方结束通知，sdk才会结束清理本次会话。

###<span id="数据发送与接收">数据发送与接收</span>
发送数据由nim\_rts\_send\_data实现，接收数据由nim\_rts\_set\_rec\_data\_cb\_func的nim\_rts\_rec\_data\_cb\_func回调监听。发送数据接口需要在通道成功建立并有2个或以上成员（包括自己）时才能发送数据。

###<span id="rts简单示例">rts简单示例</span>
下面是一个tcp通道的简单示例，所有回调接口建议post到UI线程处理

	static std::string session_id_;//会话id
	//会话发起结果回调
	void RtsStartCb(int code, const char *session_id, int channel_type, const char* uid, const char *json_extension, const void *user_data)
	{
		if (code == 200)
		{
			session_id_ = session_id;
		}
	}
	//一个会话邀请通知
	void RtsStartNotifyCb(const char *session_id, int channel_type, const char* uid, const char *json_extension, const void *user_data)
	{
		if (session_id_.empty())
		{
			session_id_ = session_id;
			nim_rts_ack(session_id, channel_type, true, "", nullptr, nullptr);
		}
		else
		{
			nim_rts_ack(session_id, channel_type, false, "", nullptr, nullptr);
		}
	}
	//收到被邀请方回复的通知回调
	void RtsAckNotifyCb(const char *session_id, int channel_type, bool accept, const char *uid, const char *json_extension, const void *user_data)
	{
		if (session_id == session_id_)
		{
			if (accept)
			{
				//对方同意，sdk底层开始连接
			}
			else
			{
				session_id_.clear();
			}
		}
	}
	//连接状态的回调
	void RtsConnectNotifyCb(const char *session_id, int channel_type, int code, const char *json_extension, const void *user_data)
	{
		if (session_id == session_id_)
		{
			if (code != 200)//连接异常，挂断
			{
				nim_rts_hangup(session_id_.c_str(), "", nullptr, nullptr);
				session_id_.clear();
			}
		}
	}
	//成员变化回调
	void RtsMemberChangeCb(const char *session_id, int channel_type, int type, const char *uid, const char *json_extension, const void *user_data)
	{
		if (session_id == session_id_)
		{
			if (type == kNIMRtsMemberStatusJoined)
			{
				//成员进入，此时可以在tcp通道发送数据
			}
		}
	}
	void RtsHangupNotifyCb(const char *session_id, const char *uid, const char *json_extension, const void *user_data)
	{
		if (session_id == session_id_)
		{
			session_id_.clear();
		}
	}
	//收到会话中的数据
	void RtsAppDataCb(const char *session_id, int channel_type, const char* uid, const char* data, unsigned int size, const char *json_extension, const void *user_data)
	{
	}
	//初始化
	void init()
	{
		nim_rts_set_start_notify_cb_func(RtsStartNotifyCb, nullptr);
		nim_rts_set_ack_notify_cb_func(RtsAckNotifyCb, nullptr);
		nim_rts_set_connect_notify_cb_func(RtsConnectNotifyCb, nullptr);
		nim_rts_set_member_change_cb_func(RtsMemberChangeCb, nullptr);
		nim_rts_hangup(RtsHangupNotifyCb, nullptr);
		nim_rts_set_rec_data_cb_func(RtsAppDataCb, nullptr);
	}
	//发起一个会话
	nim_rts_start(kNIMRtsChannelTypeTcp, "account", "", RtsStartCb, nullptr);
	//发送数据
	std::string data = "test 123";
	nim_rts_send_data(session_id_.c_str(), kNIMRtsChannelTypeTcp, data.c_str(), data.size(), "");
	//挂断
	nim_rts_hangup(session_id_.c_str(), "", nullptr, nullptr);
	session_id_.clear();

## <span id="系统通知">系统通知</span>

除消息通道外，SDK 还提供系统通知这种通道用于消息之外的通知分发。目前有两种类型:内置系统通知和自定义系统通知。

所有的系统通知（包括内置系统通知和自定义系统通知）都是通过
	
	void nim_sysmsg_reg_sysmsg_cb(const char *json_extension, nim_sysmsg_receive_cb_func cb, const void *user_data);

注册后的回调通知给APP。为了保证整个程序逻辑的一致性，APP 需要针对不同类型的系统通知进行相应的操作。

###<span id="内置系统通知">内置系统通知</span>

这是由SDK 预定义的通知类型，目前仅支持几种群操作的通知，如被邀请入群，SDK 负责这些通知的持久化。

此外，SDK 提供了以下接口来获取和维护内置系统通知记录：

	查询系统消息列表(按时间逆序查询，逆序排列)
	void nim_sysmsg_query_msg_async(int limit_count, __int64 last_time, const char *json_extension, nim_sysmsg_query_cb_func cb, const void *user_data);
	
	查询未读消息数
	void nim_sysmsg_query_unread_count(const char *json_extension, nim_sysmsg_res_cb_func cb, const void *user_data);
	
	设置消息状态
	void nim_sysmsg_set_status_async(__int64 msg_id, NIMSysMsgStatus status, const char *json_extension, nim_sysmsg_res_ex_cb_func cb, const void *user_data);
	
	删除单条消息
	void nim_sysmsg_delete_async(__int64 msg_id, const char *json_extension, nim_sysmsg_res_ex_cb_func cb, const void *user_data);

	设置全部消息为已读
	void nim_sysmsg_read_all_async(const char *json_extension, nim_sysmsg_res_cb_func cb, const void *user_data);
	
	删除全部消息
	void nim_sysmsg_delete_all_async(const char *json_extension, nim_sysmsg_res_cb_func cb, const void *user_data);
	
###<span id="自定义系统通知">自定义系统通知</span>

除了内置系统通知外，SDK 也额外提供了自定义系统给开发者，方便开发者进行业务逻辑的通知。这个通知既可以由客户端发起也可以由开发者服务器发起。

客户端发起的自定义通知，该类型通知格式由开发者自定义(kNIMSysMsgKeyAttach 里)，SDK 仅负责发送、接收，支持在线或离线发送，且支持点对点通知和群通知，不做任何解析，也不会存储，因此也不会在聊天记录中体现。示例如下：
	
	typedef void(*nim_talk_send_custom_sysmsg)(const char *json_msg, const char *json_extension);
	
	void foo()
	{
		//使用场景：如正在输入状态
		nim_talk_send_custom_sysmsg func = (nim_talk_send_custom_sysmsg) GetProcAddress(hInst, "nim_talk_send_custom_sysmsg");
		//@param json_msg：统一以系统通知的形式通知，详见系统消息字段
		func(json_msg, nullptr);
	}

客户端发起的自定义通知的回执结果通过APP 预先通过

	void nim_talk_reg_receive_cb(const char *json_extension, nim_talk_receive_cb_func cb, const void *user_data);

注册的回调告知开发者。

**SDK 并不负责自定义通知的持久化，APP 需要根据自己的业务逻辑按需进行解析和持久化的工作。**

## <span id="用户帐号">用户帐号</span>
### <span id="概述">概述</span>
SDK 提供了用户帐号资料管理。以下几个接口仅当选择云信托管用户资料时有效，如果开发者不希望云信获取自己的用户数据，则需自行维护用户资料。

nim\_user\_def.h 里定义了用户信息的字段 kUInfoKeyXXX。

用户信息变更会通过注册的全用户信息变更通知回调告知APP：

	void nim_user_reg_user_info_changed_cb(const char *json_extension, nim_user_info_change_cb_func cb, const void *user_data);

###<span id="获取本地用户信息">获取本地用户信息</span>

	void nim_user_get_user_info(const char *accids, const char *json_extension, nim_user_get_user_info_cb_func cb, const void *user_data);

###<span id="获取服务器用户信息">获取服务器用户信息</span>

	void nim_user_get_user_info_online(const char *accids, const char *json_extension, nim_user_get_user_info_cb_func cb, const void *user_data);

###<span id="编辑用户资料">编辑用户资料</span>

	void nim_user_update_user_info(const char *info_json, const char *json_extension, nim_user_update_info_cb_func cb, const void *user_data);

例：

	void CallbackUpdateUInfo(int error, const char *json_extension, const void *user_data)
	{
		if (error == kNIMResSuccess)
		...
		else
		...
	}

	typedef void (*nim_user_update_user_info)(const char *info_json, const char *json_extension, nim_user_update_info_cb_func cb, const void *user_data);

	void foo()
	{
		//修改昵称
		Json::Value values;
		values[kUInfoKeyAccid] = "litianyi02";
		values[kUInfoKeyName] = "修改后的大意的昵称";

		nim_user_update_user_info func = (nim_user_update_user_info) GetProcAddress(hInst, "nim_user_update_user_info");
		func(values.toStyledString().c_str(), nullptr, &CallbackUpdateUInfo, nullptr);
	}

## <span id="用户关系托管">用户关系托管</span>
SDK 提供了用户好友关系管理，以及对用户会话的消息设置。在云信中，不是好友也允许聊天。好友关系如果不托管给云信，开发者需要自己在应用服务器维护。

添加/被添加，请求/被请求，删除/被删除的通知以及多端同步等通知通过注册的好友数据变更通知回调告知APP：

	void nim_friend_reg_changed_cb(const char *json_extension, nim_friend_change_cb_func cb, const void *user_data);

例:

	void CallbackFriendChange(NIMFriendChangeType type, const char *result_json ,const char *json_extension, const void *user_data)
	{
		switch (type)
		{
			case kNIMFriendChangeTypeRequest:
				...
				break;
			case kNIMFriendChangeTypeDel:
				...
				break;
			...
	}

	typedef void(*nim_friend_reg_changed_cb)(const char *json_extension, nim_friend_change_cb_func cb, const void *user_data);

	void foo()
	{
		nim_friend_reg_changed_cb func = (nim_friend_reg_changed_cb) GetProcAddress(hInst, "nim_friend_reg_changed_cb");
		func(nullptr, &CallbackFriendChange, nullptr);
	}

###<span id="好友关系">好友关系</span>

- 获取好友列表

		void nim_friend_get_list(const char *json_extension, nim_friend_get_list_cb_func cb, const void *user_data);

- 好友请求

	好友请求包括**请求添加好友**以及**同意/拒绝好友请求**两种。

	好友验证包括直接添加为好友(kNIMVerifyTypeAdd)，请求添加好友(kNIMVerifyTypeAsk)，并且可以携带客户端自定义消息(msg)完成附加功能，例如添加好友附言功能。

		void nim_friend_request(const char *accid, NIMVerifyType verify_type, const char *msg, const char *json_extension, nim_friend_opt_cb_func cb, const void *user_data);

	回调函数告知接口调用是否成功，以不需要验证方式的好友请求为例：

		void CallbackFriendOpt(int res_code, const char *json_extension, const void *user_data)
		{
			if (res_code == kNIMResSuccess)
				...
			else
				...	
		}

		typedef void(*nim_friend_request)(const char *accid, NIMVerifyType verify_type, const char *msg, const char *json_extension, nim_friend_opt_cb_func cb, const void *user_data);

		void foo()
		{
			nim_friend_request func = (nim_friend_request) GetProcAddress(hInst, "nim_friend_request");
			func("id", kNIMVerifyTypeAdd, nullptr, nullptr, &CallbackFriendOpt, nullptr);
		}

	如果好友验证方式为需要验证(kNIMVerifyTypeAsk)，对方收到消息之后，可以选择同意(kNIMVerifyTypeAgree)或者拒绝好友请求(kNIMVerifyTypeReject)，此时同样调用nim\_friend\_request 接口，传入拒绝对象的ID和验证回复类型即可。

- 删除好友

		void nim_friend_delete(const char *accid, const char *json_extension, nim_friend_opt_cb_func cb, const void *user_data);

- 更新资料
	
		void nim_friend_update(const char *friend_json, const char *json_extension, nim_friend_opt_cb_func cb, const void *user_data);


### <span id="黑名单">黑名单</span>
云信中，黑名单和用户关系是互相独立的，即修改用户关系不会影响黑名单关系，同时，修改黑名单也不会对用户关系进行操作。

黑名单列表有本地缓存，缓存会在手动/自动登录后与服务器自动进行同步更新。通过注册用户关系变更通知回调获取当前数据变化：

	void nim_user_reg_special_relationship_changed_cb(const char *json_extension, nim_user_special_relationship_change_cb_func cb, const void *user_data);

例：

	void CallbackUserRelationshipChanged(NIMUserSpecialRelationshipChangeType type, const char *result_json ,const char *json_extension, const void *user_data)
	{
		switch (type)
		{
			case kNIMUserSpecialRelationshipChangeTypeMarkBlack:
				//解析result_json
				break;
			case kNIMUserSpecialRelationshipChangeTypeMarkMute:
				//解析result_json
				break;
			...
		}
	}

	typedef	void (*nim_user_reg_special_relationship_changed_cb)(const char *json_extension, nim_user_special_relationship_change_cb_func cb, const void *user_data);

	void foo()
	{
		nim_user_reg_special_relationship_changed_cb func = (nim_user_reg_special_relationship_changed_cb) GetProcAddress(hInst, "nim_user_reg_special_relationship_changed_cb");
		func(nullptr, &CallbackUserRelationshipChanged, nullptr);
	}

- 加入/移除黑名单

		void nim_user_set_black(const char *accid, bool set_black, const char *json_extension, nim_user_opt_cb_func cb, const void *user_data);

	回调函数告知接口调用是否成功，例：

		void CallbackUserOpt(int res_code, const char *accid, bool opt, const char *json_extension, const void *user_data)
		{
			if (res_code == kNIMResSuccess)
    	    	...
    		else
    	    	... 
		}

		typedef	void (*nim_user_set_black)(const char *accid, bool set_black, const char *json_extension, nim_user_opt_cb_func cb, const void *user_data);

		void foo()
		{
			nim_user_set_black func = (nim_user_set_black) GetProcAddress(hInst, "nim_user_set_black");
			func("id", true, nullptr, &CallbackUserOpt, nullptr);
		}

- 获取黑名单

		void nim_user_get_mute_blacklist(const char *json_extension, nim_user_sync_muteandblacklist_cb_func cb, const void *user_data);

### <span id="消息提醒">消息提醒</span>
云信中，可以单独设置是否开启某个用户的消息提醒，即对某个用户静音。静音关系和用户关系是互相独立的，修改用户关系不会影响静音关系，同时，修改静音关系也不会对用户关系进行操作。

静音名单列表有本地缓存，缓存会在手动/自动登录后与服务器自动进行同步更新。通过注册用户关系变更通知回调获取当前数据变化：

	void nim_user_reg_special_relationship_changed_cb(const char *json_extension, nim_user_special_relationship_change_cb_func cb, const void *user_data);

- 加入/移除静音名单列表

		void nim_user_set_mute(const char *accid, bool set_mute, const char *json_extension, nim_user_opt_cb_func cb, const void *user_data);

- 获取静音名单列表

		void nim_user_get_mute_blacklist(const char *json_extension, nim_user_sync_muteandblacklist_cb_func cb, const void *user_data);

## <span id="HTTP上传与下载">HTTP下载与上传</span>   

###<span id="初始化">初始化</span>

由于SDK 依赖HTTP 模块，所以HTTP 模块的加载和释放由SDK 来管理

###<span id="下载">下载</span>

下载资源，回调函数包括了下载结果以及下载进度

	下载结果回调函数
	@param rescode：200表示下载成功
	@param file_path：下载文件的完整路径
	@param call_id：对话id
	@param res_id：消息id，与对话id一起可定位到具体的消息，然后根据rescode调整UI呈现
	typedef void(*nim_http_dl_cb_func)(int rescode, const char *file_path, const char *call_id, const char *res_id, const char *json_extension, const void *user_data);

	下载进度回调函数
	@param downloaded_size：已下载大小
	@param file_size：文件大小，单位都是字节
	typedef void(*nim_http_dl_prg_cb_func)(__int64 downloaded_size, __int64 file_size, const char *json_extension, const void *user_data);

	下载资源接口
	typedef void(*nim_http_dl_cb_func)(int rescode, const char *file_path, const char *call_id, const char *res_id, const char *json_extension, const void *user_data);
	typedef void(*nim_http_dl_prg_cb_func)(__int64 downloaded_size, __int64 file_size, const char *json_extension, const void *user_data);
	@param nos_url：资源url
	void nim_http_dl(const char *nos_url, nim_http_dl_cb_func res_cb, const void *res_user_data, nim_http_dl_prg_cb_func prg_cb, const void *prg_user_data);

###<span id="上传">上传</span>

上传资源，回调函数包括了上传结果以及上传进度

	上传结果回调函数
	@param rescode：错误码，200表示成功
	@param url：上传成功后返回的url
	typedef void(*nim_http_up_cb_func)(int rescode, const char *url, const char *json_extension, const void *user_data);

	上传进度回调函数
	@uploaded_size：已上传大小
	@file_size：文件大小
	typedef void(*nim_http_up_prg_cb_func)(__int64 uploaded_size, __int64 file_size, const char *json_extension, const void *user_data);	

	//上传资源接口
	@param local_file：待上传的文件完整路径
	void nim_http_up(const char *local_file, nim_http_up_cb_func res_cb, const void *res_user_data, nim_http_up_prg_cb_func prg_cb, const void *prg_user_data);


## <span id="SDK 开发示例">SDK 开发示例</span> 
	#include "stdafx.h"
	#include <windows.h>
	#include <cassert>
	#include <iostream>
	#include <string>
	#include "json/json.h"
	#include "nim_res_code_def.h"
	#include "nim_client_def.h"

	//初始化和卸载
	typedef bool (*nim_client_init)(const char *app_data_dir, const char *app_install_dir, const char *json_extension);
	typedef void (*nim_client_cleanup)(const char *json_extension);

	//登录和退出
	typedef void (*nim_client_login)(const char *app_key, const char *account, const char *password, const char *json_extension, nim_json_transport_cb_func cb, const void *user_data);
	typedef void (*nim_client_logout)(NIMLogoutType logout_type, const char *json_extension, nim_json_transport_cb_func cb, const void *user_data);

	//加载模块和接口
	class SdkNim
	{
	public:
		template <typename F>
		static F Function(const char* function_name)
		{
			F f = (F) ::GetProcAddress(nim_sdk_, function_name);
			assert(f);
			return f;
		}
	public:
		static bool Load()
		{
			nim_sdk_ = ::LoadLibraryW(L"nim_sdk_dll.dll");
			if( nim_sdk_ == NULL )
			{
				wprintf_s(L"Load nim_sdk_dll.dll Fail: %d", ::GetLastError());
				return false;
			}
			else
			{
				return true;
			}
		}
		static void UnLoad()
		{
			assert(nim_sdk_ != NULL);
			if( nim_sdk_ != NULL )
			{
				::FreeLibrary(nim_sdk_);
				nim_sdk_ = NULL;
			}
		}
		static bool Init()
		{
			nim_client_init f_init = Function<nim_client_init>("nim_client_init");
			return f_init("Netease", "", "");
		}
		static void UnInit()
		{
			assert(nim_sdk_ != NULL);
			if( nim_sdk_ != NULL )
			{
				nim_client_cleanup f_cleanup = Function<nim_client_cleanup>("nim_client_cleanup");
				f_cleanup("");
			}
		}
	private:
		static HINSTANCE nim_sdk_;
	};

	HINSTANCE SdkNim::nim_sdk_ = NULL;
	
	//设置简单的标记
	bool wait_for_login = false, wait_for_logout = false;
	bool login_success = false;

	//回调函数
	void OnLoginCallback(const char *json_params, const void *user_data)
	{
		printf_s("OnLoginCallback:%s", json_params);
	
		Json::Value json;
		Json::Reader reader;
		if( reader.parse(json_params, json) )
		{
			int code = json[kNIMErrorCode].asInt();
			int step = json[kNIMLoginStep].asInt();
			if( code == kNIMResSuccess )
			{
				if( step == kNIMLoginStepLogin )
				{
					login_success = true;
					wait_for_login = false;
				}
			}
			else
			{
				wait_for_login = false;
			}
		}
		else
		{
			assert(0);
			wait_for_login = false;
		}
	}

	void OnLogoutCallback(const char *json_params, const void *user_data)
	{
		printf_s("OnLogoutCallback:%s", json_params);
	
		wait_for_logout = false;
	}

	//登录
	void NimLogin()
	{
		nim_client_login f_login = SdkNim::Function<nim_client_login>("nim_client_login");
		f_login("app_key", "account", "password", "", &OnLoginCallback, NULL);
	}

	//退出
	void NimLogout()
	{
		nim_client_logout f_logout = SdkNim::Function<nim_client_logout>("nim_client_logout");
		f_logout(kNIMLogoutAppExit, "", &OnLogoutCallback, NULL);
	}

	int main(int, char**)
	{
		if( SdkNim::Load() )
		{
			if( SdkNim::Init() )
			{
				wait_for_login = true;
				NimLogin();
	
				while( wait_for_login )
				{
					::Sleep(1000);
				}
	
				if( login_success )
				{
					::Sleep(3000);
	
					wait_for_logout = true;
					NimLogout();
	
					while( wait_for_logout )
					{
						::Sleep(1000);
					}
				}
	
				SdkNim::UnInit();
			}
			else
			{
				wprintf_s(L"SdkNim::Init Fail");
			}
			SdkNim::UnLoad();
		}
		system("pause");
		return 0;
	}


## <span id="SDKC++封装层">SDK C++封装层</span>
 * [C++ 封装层](?t=pc&md=pc_cpp "target=_blank")
 
## <span id="API文档">API文档</span>
* [SDK在线文档(C)](http://dev.netease.im/doc/pc/NIMSDKAPI_C/html/files.html "target=_blank")
* [SDK在线文档(C++)](http://dev.netease.im/doc/pc/NIMSDKAPI_CPP/html/annotated.html "target=_blank")
* [HTTP在线文档](http://dev.netease.im/doc/pc/NIMHttpAPI/html/files.html "target=_blank")
* [Audio在线文档](http://dev.netease.im/doc/pc/NIMAudioAPI/html/files.html "target=_blank")

## <span id="changelog">Change Log</span>
 * [更新记录](?t=pc&md=pc_changelog "target=_blank")

## <span id="状态码">状态码</span>
 * [状态码表](?md=nim_status_code "target=_blank")
 