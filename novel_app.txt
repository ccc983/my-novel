import streamlit as st
from openai import OpenAI

# ---------- 安全读取密钥 ----------
client = OpenAI(
    api_key=st.secrets["API_KEY"],
    base_url=st.secrets["BASE_URL"]
)
MODEL = st.secrets.get("MODEL", "deepseek-ai/DeepSeek-V3")

st.set_page_config(page_title="我的私人作家", page_icon="✍️")
st.title("✍️ 我的私人小说工坊")

# ---------- 侧边栏设定 ----------
with st.sidebar:
    st.header("📖 故事设定")
    title = st.text_input("书名", value="星海归途")
    genre = st.text_input("类型", value="科幻·星际文明·热血")
    world = st.text_area("世界观", value="公元2245年，人类踏入星际大航海时代。宇宙中存在一种名为「源晶」的神秘能量，各大势力为此明争暗斗。")
    characters = st.text_area("主要人物", value="主角·陈帆：23岁，星舰学院落榜生，意外激活血脉中的远古基因。\n女主·叶琉璃：来历神秘的女战士，背负着母星毁灭的秘密。")
    style_desc = st.text_area("写作风格与去AI味要求",
        value="像一位老友在给你讲故事，口语化、接地气。长短句交错，多用断句、反问、内心独白。对话要像真人，带点口头禅和反讽。禁止使用‘然而’‘与此同时’‘不可否认的是’等书面套话。允许不完美的细节，比如环境噪音、角色小动作。整体节奏明快，金句频出。")
    outline = st.text_area("章节大纲（每行一章）", value="1.废柴的意外觉醒\n2.星际海盗的夜袭\n3.叶琉璃的条件\n4.前往深渊遗迹\n5.遗迹守护者的试炼")
    chapter_num = st.number_input("生成第几章", min_value=1, value=1, step=1)
    temperature = st.slider("文风自由度", 0.7, 1.2, 1.0, 0.05, help="越高越放飞，越低越保守")
    max_words = st.selectbox("目标字数", ["2000字", "3000字", "5000字"], index=1)

    st.divider()
    col1, col2 = st.columns(2)
    gen_btn = col1.button("✍️ 生成本章", use_container_width=True)
    next_btn = col2.button("📖 续写下一章", use_container_width=True)

    if st.button("🗑️ 清空全部内容"):
        st.session_state.novel_text = ""
        st.rerun()

# ---------- 系统提示词构建 ----------
def build_prompt(ch):
    return f"""你是一位拥有十年以上经验的网络文学大神，擅长{genre}类小说，笔名「墨渊」。
现在请根据以下设定，撰写小说《{title}》的第{ch}章。

【世界观】
{world}

【主要人物】
{characters}

【全书大纲】
{outline}

【写作要求】
- 风格：{style_desc}
- 本章字数：约{max_words}
- 情节紧凑，结尾必须有悬念或情感钩子
- 直接输出正文，不要解释，不要标题
- 绝对避免任何“AI生成感”，要让人以为这是真人作者写的"""

# ---------- 内容存储 ----------
if "novel_text" not in st.session_state:
    st.session_state.novel_text = ""

st.subheader("📄 你的小说")
content = st.text_area(
    "正文（可直接编辑）",
    value=st.session_state.novel_text,
    height=500,
    key="editor",
    label_visibility="collapsed"
)
if content != st.session_state.novel_text:
    st.session_state.novel_text = content

# ---------- 生成章节 ----------
def generate(ch, prev_text=""):
    messages = [{"role": "system", "content": build_prompt(ch)}]
    if prev_text:
        messages.append({"role": "user", "content": f"上一章结尾：\n{prev_text[-600:]}\n\n请续写第{ch}章。"})
    else:
        messages.append({"role": "user", "content": f"请开始写第{ch}章。"})

    try:
        resp = client.chat.completions.create(
            model=MODEL,
            messages=messages,
            temperature=temperature,
            max_tokens=4096
        )
        return resp.choices[0].message.content
    except Exception as e:
        return f"❌ 失败：{e}"

if gen_btn:
    with st.spinner("正在创作..."):
        new_ch = generate(chapter_num)
    if new_ch and not new_ch.startswith("❌"):
        st.session_state.novel_text += f"\n\n第{chapter_num}章\n{new_ch}"
        st.rerun()
    else:
        st.error(new_ch)

if next_btn:
    next_ch = chapter_num + 1
    with st.spinner(f"续写第{next_ch}章..."):
        new_ch = generate(next_ch, st.session_state.novel_text)
    if new_ch and not new_ch.startswith("❌"):
        st.session_state.novel_text += f"\n\n第{next_ch}章\n{new_ch}"
        st.success(f"第{next_ch}章已生成，请将章节号改为{next_ch+1}后继续")
        st.rerun()
    else:
        st.error(new_ch)