<template>
  <GlobalEvents
    v-if='isLoaded'
    :filter="(event, handler, eventName) => event.target.tagName !== 'INPUT'"
    @keydown.esc='close'
  />
  <div v-if='isNotFound' class='issue'>
    <div class='issue-header'>
      <div class='close' @click='close' />
    </div>
    <p class='not-found'>Couldn't found issue</p>
  </div>
  <div v-else class='issue'>
    <div class='issue-header' :style='headerBackgroundColor'>
      <span v-if='state' class='state' :class='{"closed": isClosed}'>{{ state }}</span>
      <span v-if='isArchived' class='archive'>Archived</span>
      <span class='repo'>{{ repositoryName }}</span>
      <span v-if='isReadonly' class='readonly' title='You do not have write access for this repository'>Read only</span>
      <ButtonIcon name='close' @click='close' style='float: right' />
    </div>
    <IssueBody>
      <div class='left'>
        <Title
          v-show='isLoaded'
          :title='title'
          :url='url'
          :number='number'
          :isEditable='isLoaded && !isReadonly'
          @save='updateTitle'
        />
        <div v-if='isLoaded' class='main-comment comment'>
          <img class='avatar' :src='fetchedIssue.author.avatarUrl' />

          <div v-if='!isEditBody' class='content'>
            <div class='header'>
              <div>
                <a class='author' :href='fetchedIssue.author.url'>{{ fetchedIssue.author.login }}</a>
                <span class='ago' :title='fetchedIssue.createdAt'>
                  commented {{ fetchedIssue.createdAgo }}
                </span>
              </div>
              <ButtonIcon v-if='!isReadonly' name='edit' @click='startEditBody' />
            </div>

            <div v-if='isBodyEmpty' class='text empty'>No description provided</div>
            <div v-else
              class='text markdown-body'
              v-html='markdown(tmpBody)'
            />
          </div>
          <MarkdownEditor
            ref='body'
            v-if='isEditBody'
            v-model='tmpBody'
            class='issue-body-editor'
            :assignable-users='assignableUsers'
            :disabled='isSubmitting'
            @submit='updateBody'
          >
            <template #actions>
              <Button
                type='outline'
                text='Cancel'
                @click='cancelEditBody'
              />
              <Button
                class='update'
                type='indigo'
                text='Update'
                :isLoading='isSubmitting'
                @click='updateBody'
              />
            </template>
          </MarkdownEditor>
        </div>

        <div v-if='isCommentLoaded' class='comments'>
          <Comment
            v-for='item in comments'
            v-bind='item'
            :is-readonly='isReadonly'
            :assignable-users='assignableUsers'
            :key='item.id'
            :repository-full-name='repositoryFullName'
            @reply='replyComment'
          />
        </div>
        <MarkdownEditor
          v-if='isCommentLoaded && !isReadonly'
          v-model='newComment'
          ref='newComment'
          :assignable-users='assignableUsers'
          :disabled='isCommentSubmitting'
          @submit='submitNewComment'
          class='comment-new'
        />
        <div v-if='isCommentLoaded && !isReadonly' class='comments-actions'>
          <Button
            v-if='canArchive'
            :isLoading='isArchiveSubmitting'
            class='button-action'
            type='outline'
            text='Archive issue'
            title='Remove from the board'
            @click='archiveIssue'
          />
          <Button
            v-if='canUnarchive'
            :isLoading='isArchiveSubmitting'
            class='button-action'
            type='outline'
            text='Unarchive issue'
            title='Send to the board'
            @click='unarchiveIssue'
          />
          <Button
            v-if='canClose'
            :isLoading='isStateSubmitting'
            class='button-action'
            type='outline'
            text='Close issue'
            @click='closeIssue'
          />
          <Button
            v-if='canReopen'
            :isLoading='isStateSubmitting'
            type='outline'
            text='Reopen issue'
            @click='reopenIssue'
            class='button-action'
          />
          <Button
            :isLoading='isCommentSubmitting'
            type='indigo'
            text='Comment'
            @click='submitNewComment'
            class='button-comment'
          />
        </div>
      </div>

      <div v-if='isLoaded'>
        <Assignees
          :assignees='assignees'
          :repositoryFullName='repositoryFullName'
          :is-readonly='isReadonly'
          @assign='toggleAssignee'
        />

        <div class='delimeter' />
        <Labels
          :labels='labels'
          :repositoryFullName='repositoryFullName'
          :is-readonly='isReadonly'
          @toggle='toggleLabel'
        />

        <div class='delimeter' />
        <Colors
          :color='color'
          :is-readonly='isReadonly'
          @toggle='toggleColor'
        />

        <div class='delimeter' />
        <Columns
          :columnId='fetchedIssue.columnId'
          :is-readonly='isReadonly'
          :is-shared='isShared'
        />
      </div>
    </IssueBody>

    <Loader v-if='isLoading || isCommentLoading' />

    <!--div>
      {{ debugStore }}
    </div-->
  </div>
</template>

<script>
import Assignees from '@/components/board/issues/assignees'
import Button from '@/components/buttons/button'
import ButtonIcon from '@/components/buttons/icon'
import Colors from '@/components/board/issues/colors'
import Columns from '@/components/board/issues/columns'
import Comment from '@/components/board/issues/comment'
import IssueBody from '@/components/board/issues/body_content'
import Labels from '@/components/board/issues/labels'
import Loader from '@/components/loader';
import Markdown from '@/utils/markdown';
import MarkdownEditor from '@/components/board/issues/markdown_editor'
import Title from '@/components/board/issues/title';
import readonlyByAssignableUsers from '@/mixins/readonly_by_assignable_users';
import { DEFAULT_COLOR } from '@/utils/colors';
import { hexRgb } from '@/utils/wcag_contrast';
import { GlobalEvents } from 'vue-global-events';
import { get, call } from 'vuex-pathify';

const INIT_COMMENT = '';
const LOST_DATA_ALERT = 'If you leave before saving, your changes will be lost. Are you sure?';

export default {
  components: {
    Assignees,
    Button,
    ButtonIcon,
    Colors,
    Columns,
    Comment,
    GlobalEvents,
    IssueBody,
    Labels,
    Loader,
    MarkdownEditor,
    Title
  },
  props: {
    isShared: { type: Boolean, default: false },
    issue: { type: Object, required: false }, // See setCurrentIssue and currentIssue.
  },
  emits: ['close'],
  mixins: [readonlyByAssignableUsers],
  data: () => ({
    newTitle: undefined,
    newComment: INIT_COMMENT,
    newBody: '',
    tmpBody: '',
    isEditBody: false,
    isSubmitting: false,
    isStateSubmitting: false,
    isArchiveSubmitting: false,
    isCommentSubmitting: false,
    assignableUsers: []
  }),
  computed: {
    isLoading: get('issue/isLoading'),
    isLoaded: get('issue/isLoaded'),
    isCommentLoading: get('issue/isCommentLoading'),
    isCommentLoaded: get('issue/isCommentLoaded'),
    isNotFound: get('issue/isNotFound'),
    fetchedIssue: get('issue'),
    comments: get('issue/comments'),
    id() { return parseInt(this.$route.params.issueId); },
    number() { return parseInt(this.$route.params.issueNumber); },
    url() { return this.issue.url || this.fetchedIssue?.url; },
    repositoryName() { return this.issue?.repositoryName || this.fetchedIssue?.repositoryName; },
    repositoryFullName() { return this.issue.repositoryFullName || this.fetchedIssue?.repositoryFullName },
    title() {
      if (this.newTitle) { return this.newTitle; }
      return this.isLoaded ? this.fetchedIssue.title : this.issue.title;
    },
    isClosed() {
      return this.isLoaded ? this.fetchedIssue.isClosed : this.issue.isClosed;
    },
    isArchived() { return this.isLoaded ? this.fetchedIssue.isArchived : false; },
    isBodyEmpty() { return this.newBody == null || this.newBody.length == 0; },
    state() {
      if (this.isClosed == null) { return null; }
      return this.isClosed ? 'closed' : 'open';
    },
    assignees() { return this.fetchedIssue.assignees; },
    labels() { return this.fetchedIssue.labels; },
    color() { return this.fetchedIssue.color; },
    canClose() { return !this.isClosed },
    canReopen() { return this.isClosed },
    canArchive() { return this.isClosed && !this.isArchived },
    canUnarchive() { return this.isArchived },

    headerBackgroundColor() {
      if (!this.isLoaded) { return; }
      if (this.fetchedIssue.color == null) { return }
      if (this.fetchedIssue.color == DEFAULT_COLOR) { return }
      const rgba = hexRgb(this.fetchedIssue.color);

      return `background-color: rgba(${rgba[0]}, ${rgba[1]}, ${rgba[2]}, 0.6)`;
    },

    // debugStoreColumns: get('board/columns'),
    // debugStoreCurrentIssue: get('board/currentIssue'),
    // debugStore() {
    //   return { currentIssue: this.debugStoreCurrentIssue, board: this.debugStoreColumns };
    // },
  },
  async created() {
    if (this.id) {
      await this.fetchIssue();
      this.addTaskEventListener();
    }
  },
  beforeUnmount() {
    this.removeTaskEventListener();
  },
  watch: {
    async id(newValue, oldValue) {
      if (newValue === oldValue) { return; }
      if (newValue == null || isNaN(newValue)) { return; }

      this.removeTaskEventListener();
      await this.fetchIssue();
      this.addTaskEventListener();
    }
  },
  methods: {
    ...call([
      'board/fetchAssignableUsers',
      'board/updateIssue',
      'board/updateIssueState',
      'board/updateBoardIssue',
      'board/silentFetch',
      'issue/fetch',
      'issue/fetchShared',
      'issue/update',
      'issue/fetchComments',
      'issue/createComment'
    ]),
    addTaskEventListener() {
      if (this.isShared) { return; }

      const body = document.getElementById('body');
      if (body.getAttribute('taskListener') !== 'true') {
        document.addEventListener('taskClick', this.taskClickHandler);
        body.setAttribute('taskListener', 'true');
      }
    },
    removeTaskEventListener() {
      if (this.isShared) { return; }

      const body = document.getElementById('body');
      if (body.getAttribute('taskListener') === 'true') {
        document.removeEventListener('taskClick', this.taskClickHandler);
        body.setAttribute('taskListener', 'false');
      }
    },
    close() {
      if (this.isEditBody) {
        if (this.newBody !== this.tmpBody) {
          if (!confirm(LOST_DATA_ALERT)) return;
          this.tmpBody = this.newBody;
        }
        this.isEditBody = false;
      } else {
        if (this.newComment !== INIT_COMMENT) {
          if (!confirm(LOST_DATA_ALERT)) return;
        }
        this.removeTaskEventListener();
        this.$emit('close');
      }
    },
    async fetchIssue() {
      if (this.isShared) { return await this.fetchSharedIssue(); }

      await this.fetch({ id: this.id });
      this.newBody = this.tmpBody = this.fetchedIssue.body;

      this.fetchComments({ id: this.id });
      // Возможно стоит перенести этот метод в
      // modules/issue fetch в commit FINISH_LOADING.
      this.updateBoardIssue({ ...this.fetchedIssue });
      const fetchAssignableUsers = await this.fetchAssignableUsers({
        repositoryFullName: this.repositoryFullName
      });
      if (fetchAssignableUsers) {
        this.assignableUsers = [...fetchAssignableUsers].sort((a, b) => (a.login > b.login) ? 1 : -1);
      }
    },
    async fetchSharedIssue() {
      await this.fetchShared({
        sharedToken: this.$route.params.token,
        id: this.id
      });
      this.newBody = this.tmpBody = this.fetchedIssue.body;
    },
    async updateTitle(newTitle) {
      if (this.isSubmitting) { return; }

      // Придумать как тобразить процесс сохранение заголовка.
      // Возможно стоит не закрывать форму редактирования, disable-ить
      // и добавлять isLoader к кнопке save.
      this.isSubmitting = true;
      this.newTitle = newTitle;
      await this.updateIssue({
        id: this.id,
        title: this.newTitle,
        columnId: this.fetchedIssue.columnId
      });

      this.isSubmitting = false;
    },
    async updateBody() {
      if (this.isSubmitting) { return; }

      this.isSubmitting = true;
      // Unnecessary here. Remove after 01.07.2022 if checkboxes will fixed.
      // this.update({ body: this.newBody });
      this.newBody = this.tmpBody;
      await this.updateIssue({
        id: this.id,
        body: this.newBody,
        columnId: this.fetchedIssue.columnId
      });
      this.isSubmitting = false;
      this.isEditBody = false;
    },
    async toggleAssignee(user) {
      if (this.isSubmitting) { return; }

      this.isSubmitting = true;
      const userIndex = this.fetchedIssue.assignees.findIndex(v => v.login === user.login);
      if (userIndex === -1) {
        this.fetchedIssue.assignees.push(user);
      } else {
        this.fetchedIssue.assignees.splice(userIndex, 1);
      }
      await this.updateIssue({
        id: this.id,
        assignees: this.fetchedIssue.assignees,
        columnId: this.fetchedIssue.columnId
      });
      this.isSubmitting = false;
    },
    async toggleLabel(label) {
      if (this.isSubmitting) { return; }

      this.isSubmitting = true;
      const labelIndex = this.fetchedIssue.labels.findIndex(v => v.name === label.name);
      if (labelIndex === -1) {
        this.fetchedIssue.labels.push(label);
      } else {
        this.fetchedIssue.labels.splice(labelIndex, 1);
      }
      await this.updateIssue({
        id: this.id,
        labels: this.fetchedIssue.labels,
        columnId: this.fetchedIssue.columnId
      });
      this.isSubmitting = false;
    },
    async toggleColor(color) {
      if (this.isSubmitting) { return; }

      this.isSubmitting = true;
      let newColor = null;
      if (this.color == null) {
        if (color !== DEFAULT_COLOR) { newColor = color; }
      } else {
        if (this.color === color) {
          newColor = DEFAULT_COLOR;
        } else {
          newColor = color;
        }
      }
      if (newColor != null) {
        this.fetchedIssue.color = newColor;
        await this.updateIssue({
          id: this.id,
          color: newColor,
          columnId: this.fetchedIssue.columnId
        });
      }
      this.isSubmitting = false;
    },
    startEditBody() {
      this.isEditBody = true;
      this.$nextTick(() => this.$refs.body.$refs.textarea.focus());
    },
    async submitNewComment() {
      if (this.newComment === INIT_COMMENT) { return; }
      if (this.isCommentSubmitting) { return; }

      this.isCommentSubmitting = true;
      await this.createComment({ body: this.newComment });
      this.newComment = INIT_COMMENT;
      this.isCommentSubmitting = false;
    },
    async closeIssue() {
      if (this.isStateSubmitting) { return; }

      this.isStateSubmitting = true;
      await this.updateIssueState({
        id: this.id,
        columnId: this.fetchedIssue.columnId,
        isClosed: true
      });
      this.isStateSubmitting = false;
    },
    async reopenIssue() {
      if (this.isStateSubmitting) { return; }

      this.isStateSubmitting = true;
      await this.updateIssueState({
        id: this.id,
        columnId: this.fetchedIssue.columnId,
        isClosed: false
      });
      this.isStateSubmitting = false;
    },
    async archiveIssue() {
      if (this.isArchiveSubmitting) { return; }

      this.isArchiveSubmitting = true;
      await this.updateIssueState({
        id: this.id,
        columnId: this.fetchedIssue.columnId,
        isArchived: true
      });
      this.isArchiveSubmitting = false;
    },
    async unarchiveIssue() {
      if (this.isArchiveSubmitting) { return; }

      this.isArchiveSubmitting = true;
      await this.updateIssueState({
        id: this.id,
        columnId: this.fetchedIssue.columnId,
        isArchived: false
      });
      const boardId = parseInt(this.$route.params.id);
      if (boardId) { this.silentFetch({ id: boardId }); }
      this.isArchiveSubmitting = false;
    },
    cancelEditBody() {
      this.isEditBody = false;
      this.tmpBody = this.newBody;
    },
    replyComment(body) {
      this.newComment = `${body}\n\n`;
      this.$nextTick(() => this.$refs.newComment.$refs.textarea.focus());
    },
    markdown(text) {
      if (this.isShared) {
        return Markdown.render(text, this.repositoryFullName);
      }

      return Markdown.render(
        text,
        this.repositoryFullName,
        /* JavaScript only. Use only ', not ". */
        (textBase64, isChecked) => {
          const prefixOld = isChecked ? "- [x] " : "- [ ] ";
          const prefixNew = isChecked ? "- [ ] " : "- [x] ";
          /* See markdown js. */
          const text = decodeURIComponent(escape(window.atob(textBase64)));
          const textOld = prefixOld + text;
          const textNew = prefixNew + text;
          const event = new CustomEvent("taskClick", { detail: {
            textOld: textOld,
            textNew: textNew
          } });
          document.dispatchEvent(event);
        }
      );
    },
    async taskClickHandler({ detail }) {
      const { textOld, textNew } = detail;
      this.newBody = this.newBody.replace(textOld, textNew);
      this.tmpBody = this.newBody;
      await this.updateIssue({
        id: this.id,
        body: this.newBody,
        columnId: this.fetchedIssue.columnId
      });
    }
  }
}
</script>

<style scoped lang='sass'>
.not-found
  font-size: 24px
  font-weight: 500
  margin: 40px 0
  text-align: center
  width: 100%

.issue-header
  border-bottom: 1px solid #c5cae9
  box-sizing: border-box
  height: 44px
  padding: 10px 14px
  position: relative

  .readonly
    background-color: #e0e0e0
    border-radius: 4px
    color: #424242
    padding: 4px 16px
    position: absolute
    top: 10px
    left: calc(50% - 50px)

  .archive
    background-color: #c5cae9
    color: #283593
    border-radius: 12px
    box-sizing: border-box
    display: inline-block
    font-size: 12px
    height: 24px
    line-height: 20px
    margin-right: 8px
    padding: 2px 8px

  .state
    background-color: #22863a
    background-image: url('../../assets/icons/issue/white_open.svg')
    background-position-y: center
    background-position-x: 4px
    background-repeat: no-repeat
    border-radius: 12px
    box-sizing: border-box
    color: #fff
    display: inline-block
    font-size: 12px
    height: 24px
    line-height: 20px
    margin-right: 8px
    padding: 2px 8px 2px 24px
    vertical-align: top

    &.closed
      background-color: #d73a49
      background-image: url('../../assets/icons/issue/white_closed.svg')

  .repo
    font-weight: 500
    line-height: 22px

.issue
  .left
    max-height: 88vh
    overflow-y: scroll
    padding-right: 1px // For .textarea-container.focused box-shadow

  .issue-body-editor
    padding-left: 50px

.comment
  position: relative
  margin-bottom: 20px

  .avatar
    width: 40px
    height: 40px
    border-radius: 20px
    background: #eee
    position: absolute
    left: 0
    top: 0

  .content
    border-radius: 4px
    border: 1px solid #e8eaf6
    margin-left: 50px
    padding: 10px

    .header
      display: flex
      justify-content: space-between
      align-items: flex-start
      margin-bottom: 2px
      min-height: 24px

      a.author
        display: inline-block
        color: #283593
        font-size: 13px
        font-weight: 500
        cursor: pointer

        &:hover
          text-decoration: underline

      .ago
        margin-left: 2px
        color: #283593
        font-size: 13px
        font-weight: 400

    .text
      font-family: Roboto
      font-size: 14px
      font-style: normal
      font-weight: 400
      line-height: 18px
      letter-spacing: 0.012em

      &.empty
        color: #9e9e9e
        font-weight: 200

.comment-new
  min-height: 100px
  margin-left: 50px

.comments-actions
  display: flex
  justify-content: flex-end

.button + .button
  margin-left: 16px

.button-action,
.button-comment
  min-width: 130px !important

button.update
  min-width: 100px

.delimeter
  border-bottom: 1px solid #e8eaf6
  padding-top: 12px
  margin-bottom: 6px
</style>
