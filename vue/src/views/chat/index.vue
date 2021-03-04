<template>
    <div class="box">
      <div class="left">
          <div class="my-info" >
              <img v-if="userImg != ''" :src="userImg" alt="">
              <span class="my-name">{{username}}</span>
          </div>

          <div class="search">
            <div class="item">
              <el-input
                @change="filterUser"
                v-model="findUser"
                placeholder="请输入内容"
                prefix-icon="el-icon-search"
              >
              </el-input>
            </div>
            <div class="list">
              <div class="item" @click="selectUser(user.userId)" :class="{'active':activeUserId==user.userId}" :user-id="user.userId" :key="user.userId" v-for="user in users">
                <div class="item-left">
                  <img :src="user.img" alt="">
                </div>
                <div class="item-center">
                  <div class="item-wrap">
                    <div class="title">
                      <div class="name">{{user.name}}</div>
                      <div class="time">{{user.time}}</div>
                    </div>
                    <div class="content">{{user.content}}</div>
                  </div>
                </div>

              </div>
            </div>
          </div>
      </div>
      <div class="right">
        <div class="show-msg">
          <div class="show-box">
            <div class="msg-list">
              <div class="msg-item " :class="{'position-right':myUserId == msg.userId}" :key="key" v-for="(msg,key) in activeMessage">
                <div class="msg-left">
                  <img :src="msg.img" alt="">
                </div>
                <div class="msg-right">
                  <div class="user-name">{{msg.name}}</div>
                  <div class="msg-content">{{msg.content}}</div>
                </div>
              </div>
            </div>
          </div>
        </div>
        <div class="send-msg">
          <el-input
            class="textareaBox"
            type="textarea"
            placeholder="请输入内容"
            v-model="sendMsgContent">
          </el-input>
          <el-button @click="doSend" class="send-button">发送(S)</el-button>
        </div>
      </div>

      <el-dialog title="设置信息" :visible.sync="dialogFormVisible" :show-close="false" :close-on-click-modal="false" :close-on-press-escape="false">
        <el-form >
          <div >
            <span :class="{'img-active':userImg==imgsrc}" v-for="imgsrc in imglist" style="width:43px;height:43px;display: inline-block;margin-right:10px;border:1px solid transparent">
              <img  @click="selectImg(imgsrc)" :src="imgsrc" style="width:41px;height:41px;" alt="">
            </span>
          </div>

          <el-form-item label="姓名" >
            <el-input v-model="username" autocomplete="off"></el-input>
          </el-form-item>
        </el-form>
        <div slot="footer" class="dialog-footer">
          <el-button type="primary" @click="setUserinfo">确 定</el-button>
        </div>
      </el-dialog>
    </div>
</template>

<script>
    export default {
        name: "index",
      data() {
        return {
          dialogFormVisible:true,
          imglist:["/static/img/1.jpg","/static/img/2.jpg","/static/img/3.jpg"],
          conn:null,
          myUserId:"",
          username:"",
          userImg:"",
          activeUserId:"",
          sendMsgContent:"",
          users:{},
          findUser:"",
          activeMessage:[

          ],
          messages:{

          }
        }
      },
      computed(){
        reversedMessage:{
          // `this` 指向 vm 实例
          return this.message.split('').reverse().join('')
        }
      },
      created() {
        this.conn = new WebSocket("ws://localhost:9999/wsPage");
        this.conn.onclose = this.close
        this.conn.onmessage = this.message
        this.dialogFormVisible = true
      },
      mounted() {

      },
      methods: {
          selectUser(userId){
            this.activeUserId = userId
            if (this.messages.hasOwnProperty(userId)){
              this.activeMessage = this.messages[userId]
            }else{
              this.activeMessage = []
            }
          },
          filterUser(){

          },
          close(){
            this.$alert('连接已经关闭', '连接关闭', {
              confirmButtonText: '确定',
              callback: action => {
              }
            });
          },
          message(evt){
            let jsonData = JSON.parse(evt.data)
            switch(jsonData.action){
              case "registor":
                if (jsonData.data.hasOwnProperty('uid')){
                  console.log("获取uid",jsonData.data['uid'])
                  this.myUserId = jsonData.data['uid']
                }
                if (jsonData.data.hasOwnProperty('users')){
                  this.users = jsonData.data['users']
                  console.log("获取users",jsonData.data['users'])
                }
                if (jsonData.data.hasOwnProperty('user')){
                  let user = jsonData.data['user']
                  if (user.userId !== this.myUserId){
                    this.users.push(user)
                  }
                }
                break
              case "unregistor":
                if (jsonData.data.hasOwnProperty('user')){
                  let user = jsonData.data['user']
                  for (let i in this.users){
                    if(this.users[i].userId == user.userId){
                      this.users.splice(i, 1);
                    }
                  }
                }
                break
              default :
                let content = jsonData.message.content
                let fromId = jsonData.message.from_user
                let img = jsonData.message.from_user_img
                let username = jsonData.message.username

                let msg = {
                  userId:fromId,
                  img:img,
                  name:username,
                  content:content,
                }
                if (this.messages.hasOwnProperty(fromId)){
                  this.messages[fromId].push(msg)
                }else{
                  this.messages[fromId] = [msg]
                }
                if(this.activeUserId == fromId){
                  this.activeMessage = this.messages[fromId]
                }
                break;

            }
          },
          sendMsg(content){
            let send = {
              "action":"message",
              "data":{},
              "message":{
                "from_user":this.myUserId,
                "from_user_img":this.userImg,
                "username":this.username,
                "to_user":this.activeUserId,
                "msg_code":0,
                "content":content,
              }
            }
            this.conn.send(JSON.stringify(send));
            let msg = {
              userId:this.myUserId,
              content:content,
              name:this.username,
              img:this.userImg,
            }
            if(this.messages.hasOwnProperty(this.activeUserId)){
              this.messages[this.activeUserId].push(msg)
            }else{
              this.messages[this.activeUserId] = [msg]
            }
            this.activeMessage = this.messages[this.activeUserId]

          },
          doSend(){
            if (this.sendMsgContent !== ""){
              this.sendMsg(this.sendMsgContent)
            }
            this.sendMsgContent = ""
          },
          selectImg(src){
              this.userImg = src;
          },
          setUserinfo(){

            if(this.userImg == ""){
              this.$message.warning("请设置头像")
              return
            }
            if(this.username == ""){
              this.$message.warning("请设置用户名")
              return
            }

            let send = {
              "action":"set",
              "data":{"username":this.username,"img":this.userImg},
              "message":{}
            }
            this.conn.send(JSON.stringify(send));
            this.dialogFormVisible = false
          }
      }
    }
</script>

<style scoped>
  .box{
    height: 100vh;
    display: flex;
  }
  .left{
    width:250px;
    background:#ECE9E7;
    height: inherit;
    overflow-y: auto;
  }
  .left .my-info{
    padding:15px 15px;
  }
  .left .my-info img{
    width:61px;
    height:61px;
    vertical-align:middle;
  }
  .left .my-info .my-name{
    display: inline-block;
    padding-left:10px;
  }

  .left .item{
    padding:15px 15px;
    display: flex;
  }
  .item-left{
    width:45px;
    height: 41px;
    display: inline-block;
  }
  .item-left img{
    width:41px;
    height: 100%;
  }
  .item-center{
    flex:1
  }
  .item-center .title{
    display: flex;
  }
  .item-center .title .name{
    font-size: 14px;
    color:#000000;
    flex:1;
  }
  .item-center .title .time{
    width:44px;
    color:#B9A899;
    font-size: 12px;
    text-align: right;
  }
  .item-center .content{
    font-size: 12px;
    color:#999999;
    margin-top:10px;
    overflow: hidden;
    text-overflow:ellipsis;
    white-space: nowrap;
  }
  .list .item:hover{
    background:rgb(220,218,218)
  }
  .right{
    flex: 1;
    background:#F5F5F5;
    box-sizing: border-box;
    display: flex;
    flex-direction: column;
  }
  .list .item.active{
    background:#C8C6C6;
  }
  .right .show-msg{
    flex:1;
    background:#F5F5F5;
    overflow: hidden;
  }
  .right {
    overflow-y: auto;
  }
  .right .send-msg{
    height:124px;
    background:#FFFFFF;
  }
  .send-button{
    float:right;
    margin-right:20px;
  }
  .send-button:hover{
    background:#129611;
    color:white;
  }
  .msg-list{
    padding:15px 0px;
    overflow-y: auto;
    height: 100%;
  }
  .show-box{
    height: 100%;
  }
  .msg-item{
    display: flex;
    margin-top:20px;
    flex-direction: row;
  }

  .msg-item.position-right{
    flex-direction: row-reverse;
  }

  .msg-item .msg-left{
    width:72px;
  }
  .msg-item .msg-left img{
    width:41px;
    height:41px;
    margin-left:20px;
    margin-right:11px;
  }
  .msg-item .msg-right .user-name{
      font-size: 12px;
      color:#B2B2B2;
      margin-bottom:5px;
  }
  .msg-item.position-right .user-name{
    text-align: right;
  }
  .msg-item .msg-right .msg-content{
    font-size: 14px;
    color:#232323;
    background:white;
    padding:5px;
    max-width:325px;
  }
  .img-active{
    border:1px solid red !important;
  }
</style>
<style>
  .textareaBox .el-textarea__inner{
    height: 80px !important;
    border:none;
    resize: none;
  }
</style>
