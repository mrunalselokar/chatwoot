<template>
  <li v-if="hasAttachments || data.content" :class="alignBubble">
    <div :class="wrapClass">
      <div v-tooltip.top-start="sentByMessage" :class="bubbleClass">
        <bubble-text
          v-if="data.content"
          :message="message"
          :is-email="isEmailContentType"
          :readable-time="readableTime"
        />
        <span
          v-if="isPending && hasAttachments"
          class="chat-bubble has-attachment agent"
        >
          {{ $t('CONVERSATION.UPLOADING_ATTACHMENTS') }}
        </span>
        <div v-if="!isPending && hasAttachments">
          <div v-for="attachment in data.attachments" :key="attachment.id">
            <bubble-image
              v-if="attachment.file_type === 'image'"
              :url="attachment.data_url"
              :readable-time="readableTime"
            />
            <audio v-else-if="attachment.file_type === 'audio'" controls>
              <source :src="attachment.data_url" />
            </audio>
            <bubble-file
              v-else
              :url="attachment.data_url"
              :readable-time="readableTime"
            />
          </div>
        </div>

        <bubble-actions
          :id="data.id"
          :sender="data.sender"
          :is-a-tweet="isATweet"
          :is-email="isEmailContentType"
          :is-private="data.private"
          :message-type="data.message_type"
          :readable-time="readableTime"
          :source-id="data.source_id"
        />
      </div>
      <spinner v-if="isPending" size="tiny" />
      <a
        v-if="isATweet && isIncoming && sender"
        class="sender--info"
        :href="twitterProfileLink"
        target="_blank"
        rel="noopener noreferrer nofollow"
      >
        <woot-thumbnail
          :src="sender.thumbnail"
          :username="sender.name"
          size="16px"
        />
        <div class="sender--available-name">
          {{ sender.name }}
        </div>
      </a>
    </div>
    <div class="context-menu-wrap">
      <context-menu
        v-if="isBubble && !isMessageDeleted"
        :is-open="showContextMenu"
        :show-copy="hasText"
        :menu-position="contextMenuPosition"
        @toggle="handleContextMenuClick"
        @delete="handleDelete"
        @copy="handleCopy"
      />
    </div>
  </li>
</template>
<script>
import copy from 'copy-text-to-clipboard';

import messageFormatterMixin from 'shared/mixins/messageFormatterMixin';
import timeMixin from '../../../mixins/time';
import BubbleText from './bubble/Text';
import BubbleImage from './bubble/Image';
import BubbleFile from './bubble/File';
import Spinner from 'shared/components/Spinner';
import ContextMenu from 'dashboard/modules/conversations/components/MessageContextMenu';

import { isEmptyObject } from 'dashboard/helper/commons';
import alertMixin from 'shared/mixins/alertMixin';
import contentTypeMixin from 'shared/mixins/contentTypeMixin';
import BubbleActions from './bubble/Actions';
import { MESSAGE_TYPE, MESSAGE_STATUS } from 'shared/constants/messages';
import { generateBotMessageContent } from './helpers/botMessageContentHelper';
import { stripStyleCharacters } from './helpers/EmailContentParser';

export default {
  components: {
    BubbleActions,
    BubbleText,
    BubbleImage,
    BubbleFile,
    ContextMenu,
    Spinner,
  },
  mixins: [alertMixin, timeMixin, messageFormatterMixin, contentTypeMixin],
  props: {
    data: {
      type: Object,
      required: true,
    },
    isATweet: {
      type: Boolean,
      default: false,
    },
  },
  data() {
    return {
      showContextMenu: false,
    };
  },
  computed: {
    message() {
      const botMessageContent = generateBotMessageContent(
        this.contentType,
        this.contentAttributes,
        {
          noResponseText: this.$t('CONVERSATION.NO_RESPONSE'),
          csat: {
            ratingTitle: this.$t('CONVERSATION.RATING_TITLE'),
            feedbackTitle: this.$t('CONVERSATION.FEEDBACK_TITLE'),
          },
        }
      );

      const {
        email: {
          html_content: { full: fullHTMLContent, reply: replyHTMLContent } = {},
        } = {},
      } = this.contentAttributes;

      if ((replyHTMLContent || fullHTMLContent) && this.isIncoming) {
        let contentToBeParsed = replyHTMLContent || fullHTMLContent || '';
        const parsedContent = stripStyleCharacters(contentToBeParsed);
        if (parsedContent) {
          return parsedContent;
        }
      }
      return (
        this.formatMessage(this.data.content, this.isATweet) + botMessageContent
      );
    },
    contentAttributes() {
      return this.data.content_attributes || {};
    },
    sender() {
      return this.data.sender || {};
    },
    contentType() {
      const {
        data: { content_type: contentType },
      } = this;
      return contentType;
    },
    twitterProfileLink() {
      const additionalAttributes = this.sender.additional_attributes || {};
      const { screen_name: screenName } = additionalAttributes;
      return `https://twitter.com/${screenName}`;
    },
    alignBubble() {
      const { message_type: messageType } = this.data;
      const isCentered = messageType === MESSAGE_TYPE.ACTIVITY;
      const isLeftAligned = messageType === MESSAGE_TYPE.INCOMING;
      const isRightAligned =
        messageType === MESSAGE_TYPE.OUTGOING ||
        messageType === MESSAGE_TYPE.TEMPLATE;

      return {
        center: isCentered,
        left: isLeftAligned,
        right: isRightAligned,
        'has-context-menu': this.showContextMenu,
        'has-tweet-menu': this.isATweet,
      };
    },
    readableTime() {
      return this.messageStamp(
        this.contentAttributes.external_created_at || this.data.created_at,
        'LLL d, h:mm a'
      );
    },
    isBubble() {
      return [0, 1, 3].includes(this.data.message_type);
    },
    isIncoming() {
      return this.data.message_type === MESSAGE_TYPE.INCOMING;
    },
    hasAttachments() {
      return !!(this.data.attachments && this.data.attachments.length > 0);
    },
    hasImageAttachment() {
      if (this.hasAttachments && this.data.attachments.length > 0) {
        const { attachments = [{}] } = this.data;
        const { file_type: fileType } = attachments[0];
        return fileType === 'image';
      }
      return false;
    },
    hasText() {
      return !!this.data.content;
    },
    isMessageDeleted() {
      return this.contentAttributes.deleted;
    },
    sentByMessage() {
      const { sender } = this;
      return this.data.message_type === 1 && !isEmptyObject(sender)
        ? {
            content: `${this.$t('CONVERSATION.SENT_BY')} ${sender.name}`,
            classes: 'top',
          }
        : false;
    },
    wrapClass() {
      return {
        wrap: this.isBubble,
        'activity-wrap': !this.isBubble,
        'is-pending': this.isPending,
      };
    },
    bubbleClass() {
      return {
        bubble: this.isBubble,
        'is-private': this.data.private,
        'is-image': this.hasImageAttachment,
        'is-text': this.hasText,
        'is-from-bot': this.isSentByBot,
      };
    },
    isPending() {
      return this.data.status === MESSAGE_STATUS.PROGRESS;
    },
    isSentByBot() {
      if (this.isPending) return false;
      return !this.sender.type || this.sender.type === 'agent_bot';
    },
    contextMenuPosition() {
      const { message_type: messageType } = this.data;
      return messageType ? 'right' : 'left';
    },
  },
  methods: {
    handleContextMenuClick() {
      this.showContextMenu = !this.showContextMenu;
    },
    async handleDelete() {
      const { conversation_id: conversationId, id: messageId } = this.data;
      try {
        await this.$store.dispatch('deleteMessage', {
          conversationId,
          messageId,
        });
        this.showAlert(this.$t('CONVERSATION.SUCCESS_DELETE_MESSAGE'));
        this.showContextMenu = false;
      } catch (error) {
        this.showAlert(this.$t('CONVERSATION.FAIL_DELETE_MESSSAGE'));
      }
    },
    handleCopy() {
      copy(this.data.content);
      this.showAlert(this.$t('CONTACT_PANEL.COPY_SUCCESSFUL'));
      this.showContextMenu = false;
    },
  },
};
</script>
<style lang="scss">
.wrap {
  > .bubble {
    &.is-image {
      padding: 0;
      overflow: hidden;

      .image {
        max-width: 32rem;
        padding: var(--space-micro);

        > img {
          border-radius: var(--border-radius-medium);
        }
      }
    }

    &.is-image.is-text > .message-text__wrap {
      max-width: 32rem;
      padding: var(--space-small) var(--space-normal);
    }

    &.is-private .file.message-text__wrap {
      .ion-document-text {
        color: var(--w-400);
      }
      .text-block-title {
        color: #3c4858;
      }
      .download.button {
        color: var(--w-400);
      }
    }

    &.is-private.is-text > .message-text__wrap .link {
      color: var(--w-700);
    }
    &.is-private.is-text > .message-text__wrap .prosemirror-mention-node {
      font-weight: var(--font-weight-black);
      background: none;
      border-radius: var(--border-radius-small);
      padding: 0;
      color: var(--color-body);
      text-decoration: underline;
    }

    &.is-from-bot {
      background: var(--v-400);
      .message-text--metadata .time {
        color: var(--v-50);
      }
    }
  }

  &.is-pending {
    position: relative;
    opacity: 0.8;

    .spinner {
      position: absolute;
      bottom: var(--space-smaller);
      right: var(--space-smaller);
    }

    > .is-image.is-text.bubble > .message-text__wrap {
      padding: 0;
    }
  }
}

.sender--info {
  align-items: center;
  color: var(--b-700);
  display: inline-flex;
  padding: var(--space-smaller) 0;

  .sender--available-name {
    font-size: var(--font-size-mini);
    margin-left: var(--space-smaller);
  }
}

.button--delete-message {
  visibility: hidden;
}

li.left,
li.right {
  display: flex;
  align-items: flex-end;

  &:hover .button--delete-message {
    visibility: visible;
  }
}

li.left.has-tweet-menu .context-menu {
  margin-bottom: var(--space-medium);
}

li.right .context-menu-wrap {
  margin-left: auto;
}

li.right {
  flex-direction: row-reverse;
  justify-content: flex-end;
}

.has-context-menu {
  background: var(--color-background);
  .button--delete-message {
    visibility: visible;
  }
}

.context-menu {
  position: relative;
}
</style>
