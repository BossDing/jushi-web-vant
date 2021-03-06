<template xmlns:v-solt="http://www.w3.org/1999/xhtml" xmlns:v-slot="http://www.w3.org/1999/xhtml">
  <div id="articleDetails" style="height: 100%;" class="webfont">
    <van-nav-bar
      title="文章详情页"
      left-arrow
      @click-left="$router.back(-1)"
    />
    <div @click="disableComment" style="padding: 10px 8px 0 8px;height: 100%">
      <van-row type="flex" gutter="8">
        <van-col>
          <JushiImg
            imgFit="fill"
            :imgWidth="50"
            :imgHeight="50"
            :imgUrl="article.sysUser.imageUrl"
            errorUrl="img/avatar/defaultAvatar.png"
          />
        </van-col>
        <van-col>
          <div>
            <span>{{article.sysUser.username}}</span>
          </div>
        </van-col>
      </van-row>
      <van-row type="flex" gutter="8">
        <van-col>
          <p style="font-size: 30px;">
            {{article.title}}
          </p>
        </van-col>
      </van-row>
      <van-row gutter="8">
        <van-col span="24">
          <p v-html="article.content" style="font-size: 14px;">
          </p>
        </van-col>
      </van-row>
      <div>
        <commentList ref="commentRef" url="/api/web/comment/queryPageArticle" :query="CommentQuery"
                     style="height: 100%"/>
      </div>
    </div>

    <div
      class="fixedComment">
      <van-row type="flex" justify="center" align="center">
        <van-col :span="commentLeftSpan">
          <van-row type="flex" align="center">
            <van-col v-show="isComment">
              <van-icon @click.stop="showEmoji" :name="emojiIcon" size="20"/>
            </van-col>
            <van-col span="24">
              <form action="/">
                <van-search
                  @focus="enableComment"
                  :clearable="false"
                  shape="round"
                  v-model="comment.content"
                  placeholder="请输入评论"
                  @search="commentEnter"
                >
                  <template v-slot:left-icon>
                    <span></span>
                  </template>
                </van-search>
              </form>
            </van-col>
          </van-row>
        </van-col>
        <van-col :span="commentRightSpan" v-show="!isComment">
          <van-row type="flex" justify="end" align="center">
            <van-col span="8" :style="article | articleLikeColor(user)">
              <div @click="likeArticle">
                <font-awesome-icon style="font-size: 22px" :icon="['far', 'thumbs-up']"/>
                <span>
                {{article.likeCount==null?0:article.likeCount}}
              </span>
              </div>
            </van-col>
            <van-col span="8" style="color: #FF976A;">
              <font-awesome-icon style="font-size: 22px" :icon="['far', 'comment']" size="lg"/>
              <span>
                {{article.commentCount==null?0:article.commentCount}}
              </span>
            </van-col>
          </van-row>
        </van-col>
        <van-col :span="commentRightSpan" v-show="isComment" style="height: 100%;">
          <van-row type="flex" justify="space-around" align="center">
            <van-col span="12" style="color: #ED6A0C;">
              <font-awesome-icon style="font-size: 22px" @click="commentEnter"
                                 :icon="['far', 'paper-plane']"
                                 size="lg"/>
            </van-col>
          </van-row>
        </van-col>
      </van-row>
      <div>
        <VEmojiPicker :pack="pack" @select="selectEmoji" :style="{display: emojiDisplay}"/>
      </div>
    </div>

  </div>
</template>

<script>
    import JushiImg from '@/components/Img/JushiImg'
    import {mapState} from 'vuex'
    // 评论list组件
    import commentList from '@/components/Comment/CommentList'

    // emoji表情插件
    import VEmojiPicker from 'v-emoji-picker'
    import packData from 'v-emoji-picker/data/emojis.json'

    export default {
        name: 'ArticleDetails',
        data () {
            return {
                article: {
                    sysUser: {}
                },
                CommentQuery: {
                    page: 0,
                    size: 9,
                    articleId: '',
                    order: 'createTime'
                },
                comment: {
                    content: ''
                },
                pack: packData,
                emojiDisplay: 'none',
                emojiIcon: 'smile-o',
                // 是否评论 (评论显示状态控制)
                isComment: false,
                commentLeftSpan: 13,
                commentRightSpan: 11,
                isLikeLoading: false

            }
        },
        computed: {
            ...mapState({
                user: state => state.modules.user
            })
        },
        methods: {
            /**
             *  给文章点赞
             */
            likeArticle () {
                if (this.isLikeLoading) {
                    this.$toast('点的太快了')
                    return
                }
                if (this.user.info.id == null) {
                    // 未登录
                    this.$dialog.confirm({
                        message: '还没有登录,不能点赞哦',
                        confirmButtonText: '登录',
                        cancelButtonText: '暂不登录'
                    }).then(() => {
                        this.$router.push('/login')
                    }).catch(() => {
                        // on cancel
                    })
                    return
                }
                this.isLikeLoading = true
                this.$axios.post('/api/web/article/like', {
                    articleId: this.article.id,
                    userId: this.user.info.id
                }).then(res => {
                    this.isLikeLoading = false
                    if (res.flag) {
                        this.article.likeCount = this.article.likeCount == null ? 0 : this.article.likeCount + 1
                        this.$toast.success(res.message)
                        this.$store.commit('modules/user/addLikeArticle', Object.assign({}, this.article))
                    } else {
                        this.article.likeCount = this.article.likeCount == null ? 0 : this.article.likeCount - 1
                        this.$toast.success(res.message)
                        this.$store.commit('modules/user/removeLikeArticle', Object.assign({}, this.article))
                    }
                })
            },
            /**
             * 评论回车
             */
            commentEnter () {
                if (this.user.info.id == null) {
                    // 未登录
                    this.$dialog.confirm({
                        message: '抱歉你还没有登录,不能发起评论',
                        confirmButtonText: '登录',
                        cancelButtonText: '暂不登录'
                    }).then(() => {
                        this.$router.push('/login')
                    }).catch(() => {
                        // on cancel
                    })
                    return
                }
                let content = this.comment.content
                if (content == null || content === '') {
                    this.$toast('请输入评论')
                    return
                }
                const userId = this.user.info.id
                const articleId = this.article.id
                this.$axios.post('/api/web/comment/issueComment', {
                    articleId: articleId,
                    sysUserId: userId,
                    content: content
                }).then(res => {
                    if (res.flag) {
                        this.comment.content = ''
                        this.$toast.success('评论成功')
                        res.data.sysUser.username = this.user.info.username
                        this.$refs.commentRef.pushComment(res.data)
                    } else {
                        this.$toast.fail(res.message)
                    }
                })
            },
            selectEmoji (emoji) {
                this.comment.content += emoji.emoji
            },
            /**
             * 弹起表情选择
             */
            showEmoji () {
                if (this.emojiIcon === 'smile-o') {
                    this.emojiDisplay = 'inline-flex'
                    this.emojiIcon = 'other-pay'
                } else {
                    this.emojiDisplay = 'none'
                    this.emojiIcon = 'smile-o'
                }
            },
            /**
             * 开启评论显示
             */
            enableComment () {
                this.isComment = true
                this.commentLeftSpan = 18
                this.commentRightSpan = 5
            },
            /**
             * 关闭讨论状态显示
             */
            disableComment () {
                this.isComment = false
                this.commentLeftSpan = 13
                this.commentRightSpan = 11
                this.emojiDisplay = 'none'
                this.emojiIcon = 'smile-o'
            }
        },
        created () {
            const articleId = this.$route.query.articleId
            this.CommentQuery.articleId = articleId
            this.$axios.get(`/api/web/article/${articleId}`)
                .then(res => {
                    this.article = res
                    this.$store.commit('modules/article/setArticle', res)
                })
        },
        filters: {
            articleLikeColor: (article, user) => {
                const color = {
                    color: '#FF976A'
                }
                if (user.info.likeArticles != null) {
                    let like = user.info.likeArticles.filter(item => {
                        return item.id === article.id
                    })
                    if (like.length > 0) {
                        color.color = '#1989FA'
                    }
                }
                return color
            }
        },
        components: {
            JushiImg,
            VEmojiPicker,
            commentList
        }
    }
</script>

<style lang="scss">
  .fixedComment {
    position: fixed;
    bottom: 0;
    padding: 0px 0px 0px 8px;
    background-color: rgb(255, 255, 255);
    width: 100%;
    display: block;
    color: #ED6A0C;

    .van-search {
      border-radius: 100PX;
      background-color: #F2F3F5;
      padding: 5px 5px;
    }
  }

  [class*="van-hairline"]::after {
    border: none;
  }

  #default-comment {
    margin-top: 8px;
  }
</style>
