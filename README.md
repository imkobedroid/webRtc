感谢bridge提供的思路，继续二次开发完成了音频的处理  需要服务端代码可联系


WebRtc与信令服务器通信的流程分析：
### webrtc
WebRTC，即Web Real-Time Communication，web实时通信技术。简单地说就是在web浏览器里面引入实时通信，包括音视频通话等。WebRTC实现了基于网页的语音对话或视频通话，目的是无插件实现web端的实时通信的能力。WebRTC提供了视频会议的核心技术，包括音视频的采集、编解码、网络传输、展示等功能，并且还支持跨平台，包括linux、windows、mac、android等。

> 注意：本demo下需要自己配置node.js服务器进行验证，在本地搭建的时候需要将项目中的信令服务器地址修改为本机的ip地址

### 流程分析
> 注意：demo中分客户端 ClientA 和ClientB（本demo是通过循环一一建立连接，如下描述的单个建立连接的过程）



1.ClientA当点击进入的时候会将房间的id传递给信令服务器

```
sendMessage("createAndJoinRoom",message);
```




2.信令服务器收到消息后会回调到接口里面，此接口为：createdListener

```
返回的参数有：
socket id（用于建立webrtc连接用的标志）
room id  （房间id）
peer     （其他客户端的数据）
```




3.在接口中拿到信令服务器返回的东西后开始遍历peers，如果这个时候集合为空表示自己是房间的创造者，并且处于等到状态，如果集合不为空则开始遍历这个集合并与他们一一建立通过方法getOrCreateRtcConnect进行webrtc连接（建立连接是通过服务器返回的SocketId来进行连接的）并且一一给他们发送offer

```
for (int i = 0; i < peers.length(); i++) {
    JSONObject otherPeer = peers.getJSONObject(i);
    String otherSocketId = otherPeer.getString("id");
    //创建WebRtcPeerConnection
    Peer pc = getOrCreateRtcConnect(otherSocketId);
    //通过getOrCreateRtcConnect建立的连接后再通过pc.getPc().addTrack(localVideoTrack)设置视频流！并将这个链接保存起来
    //创建offer
    pc.getPc().createOffer(pc,sdpMediaConstraints);
}

```



4.offer的创建成功会回调到下面这个方法里面：
onCreateSuccess(SessionDescription sdp)
这个时候在回调方法里面通过前面的roomid socketid等等信息创建sdp信息并再次发送给信令服务器，让信令服务器把这些消息传递给对方
具体流程如下：


```
调用PeerConnection的CreateOffer方法创建一个用于offer的SDP对象，SDP对象中保存当前音视频的相关参数。a通过PeerConnection的SetLocalDescription方法将该SDP对象保存起来并且构建信息再此发送给信令服务器，这个时候发送了offer

//设置本地LocalDescription
pc.setLocalDescription(Peer.this, sdp);
//构建信令数据
try {
    JSONObject message = new JSONObject();
    message.put("from",webRtcClient.getSocketId());
    message.put("to",id);
    message.put("room",webRtcClient.getRoomId());
    message.put("sdp",sdp.description);
    //向信令服务器发送信令
    webRtcClient.sendMessage(type,message);

```



5.当信令服务器接受到上面发送过来的offer的时候再回调offerListener，其他客户端一一接收到ClientA发送过的offer SDP对象，通过PeerConnection的SetRemoteDescription方法将其保存起来，这时再设置自己的视频流，并调用PeerConnection的CreateAnswer方法创建一个应答的SDP对象answer，通过PeerConnection的SetLocalDescription的方法保存该 Answer SDP对象并将它通过信令服务器发。

```
//获取id
String fromId = data.getString("from");
//获取peer
Peer pc = getOrCreateRtcConnect(fromId);
//构建RTCSessionDescription参数
SessionDescription sdp = new SessionDescription(
        SessionDescription.Type.fromCanonicalForm("offer"),
        data.getString("sdp")
);
//设置远端setRemoteDescription
pc.getPc().setRemoteDescription(pc,sdp);
//设置answer
pc.getPc().createAnswer(pc,sdpMediaConstraints);
```





6.当信令服务器再次接受到信息时候会回调answerListener,a接收到发送过来的应答SDP对象，将其通过PeerConnection的SetRemoteDescription方法保存起来

```
//获取id
String fromId = data.getString("from");
//获取peer
Peer pc = getOrCreateRtcConnect(fromId);
//构建RTCSessionDescription参数
SessionDescription sdp = new SessionDescription(
        SessionDescription.Type.fromCanonicalForm("answer"),
        data.getString("sdp")
);
//设置远端setRemoteDescription
pc.getPc().setRemoteDescription(pc,sdp);
```




7.在SDP信息的offer/answer流程中，ClientA和ClientB已经根据SDP信息创建好相应的音频Channel和视频Channel并开启Candidate数据的收集，Candidate数据可以简单地理解成Client端的IP地址信息（本地IP地址、公网IP地址、Relay服务端分配的地址）


 8.这个时候ClientA和ClientB开始使用穿透技术获取自己的公网地址等等信息，各自获取到后会回调到方法onIceCandidate里面并通过webRtcClient.sendMessage("candidate",message)方法将ip等信息发送给信令服务器

```
//新ice地址被找到触发
@Override
public void onIceCandidate(IceCandidate iceCandidate) {
    Log.d(TAG,"onIceCandidate "+iceCandidate.sdpMid);
    try {
        //构建信令数据
        JSONObject message = new JSONObject();
        message.put("from",webRtcClient.getSocketId());
        message.put("to",id);
        message.put("room",webRtcClient.getRoomId());
        //candidate参数
        JSONObject candidate = new JSONObject();
        candidate.put("sdpMid",iceCandidate.sdpMid);
        candidate.put("sdpMLineIndex",iceCandidate.sdpMLineIndex);
        candidate.put("sdp",iceCandidate.sdp);
        message.put("candidate",candidate);
        //向信令服务器发送信令
        webRtcClient.sendMessage("candidate",message);
    } catch (JSONException e) {
        e.printStackTrace();
    }
}

```



9.信令服务器接受到后会回调到接口candidateListener

```
//获取id
String fromId = data.getString("from");
//获取peer
Peer pc = getOrCreateRtcConnect(fromId);
//获取candidate
JSONObject candidate = data.getJSONObject("candidate");
IceCandidate iceCandidate = new IceCandidate(
        candidate.getString("sdpMid"), //描述协议id
        candidate.getInt("sdpMLineIndex"),//描述协议的行索引
        candidate.getString("sdp")//描述协议
);
//添加远端设备路由描述
pc.getPc().addIceCandidate(iceCandidate);
```




 10.这样ClientA和ClientB就已经建立了音视频传输的P2P通道，ClientB接收到ClientA传送过来的音视频流，会通过PeerConnection的OnAddStream回调接口返回一个标识ClientA端音视频流的MediaStream对象，在ClientB端渲染出来即可。同样操作也适应ClientB到ClientA的音视频流的传输，本demo使用onTrack回调代替了这个渲染的过程

```
public interface RtcListener {

    //远程音视频流加入 Peer通道
    void onAddRemoteStream(String peerId,VideoTrack videoTrack);

    //远程音视频流移除 Peer通道销毁
    void onRemoveRemoteStream(String peerId);
}


//用这个方法代替了OnAddStream方法进行了视频流的渲染
@Override
public void onTrack(RtpTransceiver transceiver) {
    MediaStreamTrack track = transceiver.getReceiver().track();
    Log.d(TAG,"onTrack "+ track.id());
    if (track instanceof VideoTrack) {
        webRtcClient.getRtcListener().onAddRemoteStream(id,(VideoTrack)track);
    }
}
```



# 接口说明

通过Socket.Io进行数据交互，Json格式

----------------------------------------Client To Server----------------------------------------

1：事件名：createAndJoinRoom    客户端通知服务器创建并加入room中，若room已存在则直接加入 {room}

    room：房间名称，字符串

2：事件名：offer 发送offer消息 {from,to,room,sdp}

    from: 发送者socket连接标识，字符串
    to:接收者socket连接标识，字符串
    room：房间名称，字符串
    sdp：发送者设备sdp描述，字符串

3：事件名：answer 发送answer消息 {from,to,room,sdp}

    from: 发送者socket连接标识，字符串
    to:接收者socket连接标识，字符串
    room：房间名称，字符串
    sdp：发送者设备sdp描述，字符串

5：事件名：candidate  发送candidate消息 {from,to,room,candidate{sdpMid,sdpMLineIndex,sdp}}

    from: 发送者socket连接标识，字符串
    to:接收者socket连接标识，字符串
    room：房间名称，字符串
    candidate：发送者设备candidate描述，Json类型
      sdpMid：描述协议id，字符串
      sdpMLineIndex：描述协议的行索引，字符串
      sdp：sdp描述协议，字符串
 
6：事件名：exit  发送exit消息 {from,room}

    from: 发送者socket连接标识，字符串
    room：房间名称，字符串

----------------------------------------Server To Client----------------------------------------

1：事件名：created   服务器通知客户端信令连接成功 {id,room,peers[{id}]}

    id: 当前socket连接标识，字符串
    room：房间名称，字符串
    peers：Json数组，房间其他客户端socket连接标识集合
      id：房间其他socket连接标识

2：事件名：joined   服务器通知客户端当前房间有新连接加入 {id,room}

    id: 新socket连接标识，字符串
    room：房间名称，字符串

3：事件名：offer  服务器转发offer消息 {from,to,room,sdp}

    from: 发送者socket连接标识，字符串
    to:接收者socket连接标识，字符串
    room：房间名称，字符串
    sdp：发送者设备sdp描述，字符串

4：事件名：answer  服务器转发answer消息 {from,to,room,sdp}

    from: 发送者socket连接标识，字符串
    to:接收者socket连接标识，字符串
    room：房间名称，字符串
    sdp：发送者设备sdp描述，字符串

5：事件名：candidate  服务器转发candidate消息 {from,to,room,candidate{sdpMid,sdpMLineIndex,sdp}}

    from: 发送者socket连接标识，字符串
    to:接收者socket连接标识，字符串
    room：房间名称，字符串
    candidate：发送者设备candidate描述，Json类型
       sdpMid：描述协议id，字符串
       sdpMLineIndex：描述协议的行索引，字符串
       sdp：sdp描述协议，字符串

6：事件名：exit  服务器转发exit消息 {from,room}

    from: 发送者socket连接标识，字符串
    room：房间名称，字符串
