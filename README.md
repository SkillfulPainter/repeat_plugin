# 麦麦复读插件 (MaiBot_repeat_plugin)
作者：火火火木（SkillfulPainter）

## 简介
修改了位于 **"MaiBot\src\plugins\built_in\core_actions\plugin.py"** 中内置的reply_plugin，在使用LLM生成回复之前新增了判断麦麦是否需要复读的方法。该更改在enable_repeat参数设置为false时不会对正常回复有任何影响。在判断复读不通过时会进入正常回复流程。
本人代码能力有限，希望能多向我提提意见，不胜感谢！

**判断复读的条件如下：**
1. enable_repeat参数是否为true，若是，则进行下一步
2. 麦麦是否在之前5分钟内/之前的6条消息内发过言，若是则不进行复读。若否，则进行下一步。
3. 之前5分钟内/之前的6条消息内是否有完全一致的消息，如果有，将复读这条消息，如果没有，不进行复读。检查消息的顺序为时间上由近到远。如果在六条消息内有多次不同复读，则麦麦会复读距离当前时间近的一条。

## 使用方法
# <span style="color:red">⚠️ ！！必读！！</span> ⚠️
对于0.8.0版本用户：
将这个plugin中的plugin.py文件复制到 **"MaiBot\src\plugins\built_in\core_actions"** 目录下并覆盖其中的 **plugin.py** 文件。然后删除该目录里的config.toml让麦麦在运行时生成新的配置文件。
对于更新的版本用户：
仔细检查**"MaiBot\src\plugins\built_in\core_actions"** 目录下的 **plugin.py** 文件与0.8.0版本的是否一致，若一致则可直接覆盖，若不一致，可尝试在
'''
try:
        success, reply_set = await generator_api.generate_reply(
        action_data=self.action_data,
        chat_id=self.chat_id,
        )
'''
前，execute(self)后加上如下判断：
'''
        repeat_probability = self.get_config("repeat.repeat_probability", 0)
        enable_repeat = self.get_config("repeat.enable_repeat", False)
        repeatable = False
        reply_text = ""

        if enable_repeat:
            context_messages = message_api.get_messages_by_time_in_chat(
                chat_id=self.chat_id,
                start_time=start_time - 300,  # 获取开始前5分钟内的消息
                end_time=start_time,
                limit=6,
                limit_mode="latest",
            )

            if len(context_messages) >= 2:
                for i in range(len(context_messages) - 1, 0, -1):
                    # 检查当前字典和下一个字典的"processed_plain_text"值
                    if context_messages[i].get("processed_plain_text") == context_messages[i - 1].get(
                            "processed_plain_text"):
                        repeatable = True  # 发现连续重复
                        reply_text = context_messages[i - 1].get("processed_plain_text")
                        logger.info(f"{self.log_prefix} 检测到可复读消息--{reply_text}--")
                        break
            # logger.info(f"context_messages={context_messages}")
            # logger.info(f"person_names={person_names}")

            for msg in context_messages:
                if msg.get('user_nickname') == global_config.bot.nickname:
                    repeatable = False
                    logger.info(f"{self.log_prefix} 检测到麦麦最近已经发过言，不进行复读。")
                    break

        if random.random() <= repeat_probability and repeatable and reply_text:    #判断是否复读
            try:
                pattern = r'@<([^:]+?):\d+>'
                reply_text = re.sub(pattern, r'@\1', reply_text)
                await self.send_text(content=reply_text, typing=False)
                logger.info(f"{self.log_prefix} 复读了--{reply_text}--")
                await self.store_action_info(
                    action_build_into_prompt=False,
                    action_prompt_display=f"你复读了：{reply_text}，之后不要对与其相同的消息进行复读",
                    action_done=True,
                )
                NoReplyAction.reset_consecutive_count()

            except Exception as e:
                logger.error(f"{self.log_prefix} 回复动作执行失败: {e}")
                return False, f"回复失败: {str(e)}"
            return True, reply_text

        else: #转入正常回复
            try:
                success, reply_set = await generator_api.generate_reply(
                    action_data=self.action_data,
                    chat_id=self.chat_id,
                )
'''
然后在插件末尾的
'''
class CoreActionsPlugin(BasePlugin):
'''
的config_schema中加入如下配置：
'''
"repeat": {
            "enable_repeat": ConfigField(type=bool, default=False, description="是否启用复读功能"),
            "repeat_probability": ConfigField(type=float, default=0.5, description="bot的复读几率", example=0.5),
        },
'''

# <span style="color:red">⚠️ 覆盖前一定要备份原文件！</span> ⚠️
# <span style="color:red">⚠️ 覆盖前一定要备份原文件！</span> ⚠️
# <span style="color:red">⚠️ 覆盖前一定要备份原文件！</span> ⚠️
你没备份导致原来的东西没了别来找我。
希望用这个插件的人都知道自己在干什么。

## 使用效果
![控制台输出](images/log_output.png)

![对话效果](images/chat.png)



