<template>
    <div>
        <div id="mobile" :class="{ 'has-prompt': hasPrompt }">
                <div id="mobile-head">
                    <div id="mobile-head-title">与羡辙聊天中
                    </div>
                </div>
                <div id="mobile-body">
                    <div id="mobile-body-bg"></div>
                    <div id="mobile-body-content">
                        <!--<div class="msg-time">21:40</div>-->
                        <div id="mock-msg-row" class="msg-row">
                            <div id="mock-msg" class="msg" v-html="latestMsgContent"></div>
                        </div>
                        <div class="msg-row"
                            v-for="(msg, index) in messages"
                            :key="index"
                            :class="msg.author === 'xianzhe' ? 'msg-xianzhe' : 'msg-me'">
                            <div class="msg"
                                :class="'msg-bounce-in-' + (msg.author === 'xianzhe' ? 'left': 'right')"
                                v-html="msg.content"></div>
                        </div>
                    </div>
                </div>
                <div id="mobile-foot">
                    <div id="prompt">
                        <div id="prompt-head">
                            <div class="say-something">说点什么……</div>
                            <a href="javascript:;" class="close-btn"
                                @click="togglePrompt(false)"></a>
                        </div>
                        <div id="prompt-body">
                            <ul class="responses" v-if="lastDialog">
                                <li v-for="res in lastDialog.responses">
                                    <a href="javascript:;" @click="respond(res)">{{ res.content }}</a>
                                </li>
                            </ul>
                            <div class="next-topic"
                                v-if="!lastDialog || !lastDialog.responses">
                                <ul class="topics">
                                    <li v-for="topic in nextTopics">
                                        <a href="javascript:;" @click="ask(topic)">{{ topic.brief }}</a>
                                    </li>
                                </ul>
                            </div>
                        </div>
                    </div>
                    <div id="input-hint" class="say-something"
                        @click="togglePrompt(true)"
                        :class="{'clickable': !isXianzheTyping }">
                        <span v-if="!isXianzheTyping">说点什么……</span>
                        <span v-if="isXianzheTyping">羡辙正在输入中</span>
                    </div>
                </div>
                <div id="prompt-bg" @click="togglePrompt(false)"></div>
            </div>
    </div>
</template>

<script>
import axios from 'axios'
export default {
  name:'Chat',
  data() {
      return {
        messages: [],
		dialogs: null,
		lastDialog: null,
		msgChain: Promise.resolve(),

		isXianzheTyping: false,

		// topics that user can ask
		nextTopics: [],

		hasPrompt: false,

		latestMsgContent: null,
		AUTHOR:{
			XIANZHE: 'xianzhe',
			ME: 'me'
		},
		 TYPING_MSG_CONTENT: `
        <div class="dot"></div>
        <div class="dot"></div>
        <div class="dot"></div>
    	`,
    	msgSendingHandler:null
      }
    },
   mounted() {
        axios.get('/api/getDialog').then((res) => {
            var data=res.data.data
			this.dialogs = data;
			this.nextTopics = this.dialogs.fromUser;
			this.appendDialog('0000');
        })

	 window.updateScroll = function() {
        const $chatbox = document.getElementById('mobile-body-content')
        const distance = $chatbox.scrollHeight - $chatbox.offsetHeight - $chatbox.scrollTop;
        const duration = 250;
        const startTime = Date.now();

        requestAnimationFrame(function step() {
            const p = Math.min(1, (Date.now() - startTime) / duration);
             $chatbox.scrollTop = $chatbox.scrollTop + distance * p
            p < 1 && requestAnimationFrame(step);
        });
    }
	 window.onMessageSending = function () {
        setTimeout(() => {
            // update scroll position when vue has updated ui
            updateScroll();

        });
    }
    window.delay = function(amount = 0) {
        return new Promise(resolve => {
            setTimeout(resolve, amount);
        });
    }

	window.getRandomMsg = function(messages){
		if (typeof messages === 'string' || !messages.length) {
            return messages;
        }

        const id = Math.floor(Math.random() * messages.length);
        return messages[id];
	}
  },
  methods: {
	
            appendDialog(id) {
                if (typeof id === 'object' && id.length > 0) {
                    // array of dialog ids
                    id.forEach(id => this.appendDialog(id));
                    return;
                }
                else if (id == null) {
                    // clear possible responses
                    this.lastDialog.responses = null;
                    return;
                }

                this.isXianzheTyping = true;

                const dialog = this.getDialog(id);

                getRandomMsg(dialog.details)
                    .forEach(content => {
                        this.msgChain = this.msgChain
                            .then(() => delay(700))
                            .then(() => this.sendMsg(content, this.AUTHOR.XIANZHE));
                    });

                return dialog.nextXianzhe
                    ? this.appendDialog(dialog.nextXianzhe)
                    : this.msgChain.then(() => {
                        this.lastDialog = dialog;
                        this.isXianzheTyping = false;
                    });
            },
			
            sendMsg(message, author) {
                switch (author) {
                    case 'me':
                        return this.sendUserMsg(message);
                    default:
                        return this.sendFriendMsg(message, author);
                }
            },
            sendFriendMsg(message, author) {
                const content = getRandomMsg(message);
                const length = content.replace(/<[^>]+>/g,"").length;
                const isImg = /<img[^>]+>/.test(content);
                const isTyping = length > 5 || isImg;

                const msg = {
                    author: author,
                    content: isTyping ? this.TYPING_MSG_CONTENT : content,
                    isImg: isImg
                };
                this.messages.push(msg);

                if (isTyping) {
                    this.markMsgSize(msg);
                    setTimeout(updateScroll);

                    return delay(Math.min(100 * length, 2000))
                        .then(() => {
                            return this.markMsgSize(msg, content);
                        })
                        .then(() => delay(150))
                        .then(() => {
                            msg.content = content;
                            onMessageSending();
                        });
                }

                onMessageSending();

                return Promise.resolve();
            },

            sendUserMsg(message) {
                this.messages.push({
                    author: this.AUTHOR.ME,
                    content: message
                });

                onMessageSending();

                return Promise.resolve();
            },

            markMsgSize(msg, content = null) {
                this.latestMsgContent = content || msg.content;
                
                return delay(0)
                    .then(() => {
                        Object.assign(msg);
                        this.messages = [...this.messages];
                    });
            },

            getDialog(id) {
                // only one dialog should be matched by id
                const dialogs = this.dialogs.fromXianzhe
                    .filter(dialog => dialog.id === id);
                return dialogs ? dialogs[0] : null;
            },

            getDialogFromUser(id) {
                // only one dialog should be matched by id
                const dialogs = this.dialogs.fromUser
                    .filter(dialog => dialog.id === id);
                return dialogs ? dialogs[0] : null;
            },

            togglePrompt(toShow) {
                if (this.isXianzheTyping) {
                    
                    return;
                }

                this.hasPrompt = toShow;
            },

            respond(response) {
               
                return this.say(response.content, response.nextXianzhe);
            },

            ask(fromUser) {
                const content = getRandomMsg(fromUser.details);
                return this.say(content, fromUser.nextXianzhe);
            },

            say(content, dialogId) {
                // close prompt
                this.hasPrompt = false;

                return delay(200)
                    // send user msg
                    .then(() => this.sendMsg(content, this.AUTHOR.ME))
                    .then(() => delay(300))
                    // add xianzhe's next dialogs
                    .then(() => this.appendDialog(dialogId));
            }
        }
}
</script>
