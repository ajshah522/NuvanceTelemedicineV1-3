<template>
  <div class="video-conference-container">
    <waiting-room
      v-if="showWaitingRoom"
      :state="waitingRoomState"
    />
    <call
      v-else-if="showVideoCall"
      :api-key="apiKey"
      :session-id="sessionId"
      :token="token"
    />
    <div
      v-else
      class="loading-container"
    >
      <div
        class="spinner-border"
        role="status"
        style="width: 3rem; height: 3rem;"
      >
        <span class="sr-only">Loading...</span>
      </div>
    </div>
  </div>
</template>
<script>
import { VIDEO_CALL_STATE } from "./../constants";
import WaitingRoom from "./WaitingRoom";
import Call from "./Call";

export default {
  name: "VideoCall",
  components: {
    WaitingRoom,
    Call
  },
  props: {
        originApiUrl: {
            type: String,
            default: ''
        }
    },
  data() {
    return {
      apiKey: null,
      sessionId: null,
      token: null,
      state: VIDEO_CALL_STATE.LOADING,
      waitingRoomState: null
    };
  },
  computed: {
    showWaitingRoom: function () {
      return this.state === VIDEO_CALL_STATE.WAITING;
    },
    showVideoCall: function () {
      return this.state === VIDEO_CALL_STATE.RUNNING;
    },
  },
  created() {},
  async mounted() {
      try {
        const result = (await this.$http.get(`${this.originApiUrl}/token/${this.$route.query.token}`)).data;
        const {apiKey, sessionId, token} = result;
        this.apiKey = apiKey;
        this.sessionId = sessionId;
        this.token = token;
        this.state = VIDEO_CALL_STATE.RUNNING;  
    } catch(err){
        console.log('API call error', err.response);
        if (err && err.response && err.response.data) {
            this.waitingRoomState = err.response.data;
        } else {
            this.waitingRoomState = {errCode: 5};
        }
        this.state = VIDEO_CALL_STATE.WAITING;
    }

  },
  methods: {
  }
};
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped lang="scss">
.video-conference-container {
  position: absolute;
  top: 80px;
  left: 0;
  right: 0;
  bottom: 0;
  background: #5f6062;
}
  .loading-container{
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    font-size: 2rem;
 /* global Chat, TextProcessor, otHelper */

!(function (exports) {
  'use strict';

  var usrId;

  var closeChatBtn;
  var headerChat;
  var sendMsgBtn;
  var chatMsgInput;
  var chatContainer;
  var chatContent;
  var chatForm;
  var chatParticipants = [];

  var _visibilityChanging = Promise.resolve();

  function isVisible() {
    return _visibilityChanging.then(function () {
      return Chat.visible;
    });
  }

  function setVisibility(isVisible) {
    if (isVisible) {
      addHandlers();
      return Chat.show().then(function () {
        Utils.sendEvent('chatView:shown');
        scrollTo();
        chatMsgInput.focus();
      });
    }
    removeHandlers();
    return Chat.hide().then(function () {
      Utils.sendEvent('chatView:hidden');
    });
  }

  var eventHandlers;

  function addEventsHandlers(configuredEvts) {
    eventHandlers = {
      incomingMessage: {
        name: 'chatController:incomingMessage',
        handler: function (evt) {
          var data = evt.detail.data;
          insertChatLine(data);
          isVisible().then(function (visible) {
            if (!visible) {
              Utils.sendEvent('chatView:unreadMessage', { data: data });
            }
          });
        }
      },
      presenceEvent: {
        name: 'chatController:presenceEvent',
        handler: function (evt) {
          insertChatEvent(evt.detail);
        }
      },
      messageDelivered: {
        name: 'chatController:messageDelivered',
        handler: function () {
          chatMsgInput.value = '';
        }
      },
      chatVisibility: {
        name: 'roomView:chatVisibility',
        handler: function (evt) {
          _visibilityChanging = setVisibility(evt.detail);
        },
        couldBeChanged: true
      }
    };
    Array.isArray(configuredEvts) && configuredEvts.forEach(function (aEvt) {
      var event = eventHandlers[aEvt.type];
      event && event.couldBeChanged && (event.name = aEvt.name);
    });
    Utils.addHandlers(eventHandlers);
  }

  function initHTMLElements() {
    var chatWndElem = document.getElementById('chat');
    headerChat = chatWndElem.querySelector('header');
    closeChatBtn = chatWndElem.querySelector('#closeChat');
    sendMsgBtn = chatWndElem.querySelector('#sendTxt');
    chatMsgInput = chatWndElem.querySelector('#msgText');
    chatContainer = chatWndElem.querySelector('#chatMsgs');
    chatContent = chatContainer.querySelector('ul');
    chatForm = chatWndElem.querySelector('#chatForm');
  }

  var onSendClicked = function (evt) {
    evt.preventDefault();
    if (!chatMsgInput.value.trim().length) {
      return;
    }
    document.activeElement.blur(); // Hide the virtual keyboard.
    Utils.sendEvent('chatView:outgoingMessage', {
      sender: usrId,
      time: Utils.getCurrentTime(),
      text: chatMsgInput.value.trim()
    });
  };

  var onKeyPress = function (myfield, evt) {
    var keycode;
    if (window.vent) {
      keycode = window.event.keyCode;
    } else if (evt) {
      keycode = evt.which;
    } else {
      return true;
    }
    if (keycode === 13) {
      if (evt.shiftKey === true) {
        return true;
      }
      onSendClicked(evt);
      return false;
    }
    return true;
  }.bind(undefined, chatMsgInput);

  var onSubmit = function (evt) {
    evt.preventDefault();
    return false;
  };

  var onClose = function (evt) {
    evt.preventDefault();
    evt.stopImmediatePropagation();
    _visibilityChanging = setVisibility(false);
  };

  var onToggle = function () {
    Chat.isCollapsed() ? Chat.expand() : Chat.collapse();
  };

  var onDrop = function (evt) {
    evt.preventDefault();
    evt.stopPropagation();
    return false;
  };

  // The ChatController should have the handlers and call the view for
  // doing visual work
  function addHandlers() {
    chatForm.addEventListener('keypress', onKeyPress);
    chatForm.addEventListener('submit', onSubmit);
    chatForm.addEventListener('drop', onDrop);
    closeChatBtn.addEventListener('click', onClose);
    headerChat.addEventListener('click', onToggle);
    sendMsgBtn.addEventListener('click', onSendClicked);
  }

  function removeHandlers() {
    chatForm.removeEventListener('keypress', onKeyPress);
    closeChatBtn.removeEventListener('click', onClose);
    headerChat.removeEventListener('click', onToggle);
    sendMsgBtn.removeEventListener('click', onSendClicked);
    chatForm.removeEventListener('drop', onDrop);
  }

  function insertChatEvent(data) {
    var time = (data.time || Utils.getCurrentTime()).toLowerCase();
    var item = HTMLElems.createElementAt(chatContent, 'li');
    item.classList.add('event');
    var name = data.sender || data.userName;
    var text = time + ' - ' + name + ' ' + data.text;
    insertText(item, text);
    scrollTo();
  }

  function insertText(elemRoot, text) {
    var txtElems = TextProcessor.parse(text);
    var targetElem = HTMLElems.createElementAt(elemRoot, 'p');
    txtElems.forEach(function (node) {
      switch (node.type) {
        case TextProcessor.TYPE.URL:
          HTMLElems.createElementAt(targetElem, 'a',
            { href: node.value, target: '_blank' }, node.value);
          break;
        default:
          HTMLElems.addText(targetElem, node.value);
      }
    });
  }

  function insertChatLine(data) {
    var item = HTMLElems.createElementAt(chatContent, 'li');
    var info = HTMLElems.createElementAt(item, 'p');
    if (otHelper.isMyself({ connectionId: data.senderId })) {
      item.classList.add('yourself');
    } else {
      var chatIndex = chatParticipants.indexOf(data.senderId);
      if (chatIndex === -1) {
        chatIndex = chatParticipants.push(data.senderId) - 1;
      }
      // We only have 10 colors so just get last digit.
      var participantNumber = chatIndex.toString().slice(-1);
      info.data('participant-number', participantNumber);
    }

    var time = data.time.toLowerCase();
    HTMLElems.createElementAt(info, 'span', null, time).classList.add('time');
    HTMLElems.createElementAt(info, 'span', null, data.sender || data.userName)
              .classList.add('sender');

    insertText(info, data.text);

    scrollTo();
  }

  function scrollTo() {
    chatContainer.scrollTop = chatContainer.scrollHeight;
  }

  function init(aUsrId, configuredEvts) {
    return LazyLoader.dependencyLoad([
      '/js/helpers/textProcessor.js',
      '/js/components/chat.js'
    ]).then(function () {
      initHTMLElements();
      usrId = aUsrId;
      Chat.init();
      addEventsHandlers(configuredEvts);
    });
  }

  var ChatView = {
    init: init
  };

  exports.ChatView = ChatView;
}(this));

  }
</style>
