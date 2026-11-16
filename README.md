import random
import streamlit as st

st.title("🐺 シンプル人狼ゲーム")

# ゲームの状態（データ）を保存する箱を初期化
if "phase" not in st.session_state:
    st.session_state.phase = "setup"  # setup, role_check, vote, result
    st.session_state.players = []
    st.session_state.roles = {}
    st.session_state.current_index = 0
    st.session_state.show_role = False
    st.session_state.votes = {}

# --- 1. 参加者登録フェーズ ---
if st.session_state.phase == "setup":
    st.subheader("1. 参加者を登録してください（3〜5人）")
    name = st.text_input("プレイヤー名を入力")

    if st.button("追加"):
        if name and name not in st.session_state.players:
            st.session_state.players.append(name)
            st.rerun()

    st.write("現在の参加者:", ", ".join(st.session_state.players))

    # 3人以上集まったらゲーム開始ボタンを表示
    if len(st.session_state.players) >= 3:
        if st.button("ゲーム開始！"):
            # 役職リスト作成（人狼1人、占い師1人、残りは村人）
            num_players = len(st.session_state.players)
            role_list = ["人狼", "占い師"] + ["村人"] * (num_players - 2)
            random.shuffle(role_list)

            # プレイヤーと役職を紐づけ
            st.session_state.roles = dict(
                zip(st.session_state.players, role_list)
            )
            st.session_state.phase = "role_check"
            st.rerun()

# --- 2. 役職確認フェーズ（1人ずつ画面を見る） ---
elif st.session_state.phase == "role_check":
    players = st.session_state.players
    idx = st.session_state.current_index
    current_player = players[idx]

    st.subheader(f"2. 役職確認（{idx + 1} / {len(players)}人目）")
    st.info(f"端末を **{current_player}** さんに渡してください。")

    if not st.session_state.show_role:
        if st.button("自分の役職を見る"):
            st.session_state.show_role = True
            st.rerun()
    else:
        my_role = st.session_state.roles[current_player]
        st.success(f"あなたの役職は **【{my_role}】** です！")

        # 占い師の簡易アクション
        if my_role == "占い師":
            target = st.selectbox(
                "占う相手を選んでください",
                [p for p in players if p != current_player],
            )
            if st.button("占う"):
                target_role = st.session_state.roles[target]
                is_wolf = "人狼です！" if target_role == "人狼" else "人狼ではありません。"
                st.warning(f"占い結果: {target} さんは **{is_wolf}**")

        if st.button("確認完了（次の人へ）"):
            st.session_state.show_role = False
            if idx + 1 < len(players):
                st.session_state.current_index += 1
            else:
                st.session_state.phase = "vote"
                st.session_state.current_index = 0
            st.rerun()

# --- 3. 議論＆投票フェーズ ---
elif st.session_state.phase == "vote":
    players = st.session_state.players
    idx = st.session_state.current_index
    current_player = players[idx]

    st.subheader("3. 昼の議論 ＆ 投票")
    st.write("全員で議論した後、1人ずつ追放したい人に投票してください。")
    st.info(f"**{current_player}** さんの投票番です。")

    vote_target = st.selectbox(
        "追放する人を選択", [p for p in players if p != current_player]
    )

    if st.button("投票する"):
        st.session_state.votes[vote_target] = (
            st.session_state.votes.get(vote_target, 0) + 1
        )
        if idx + 1 < len(players):
            st.session_state.current_index += 1
        else:
            st.session_state.phase = "result"
        st.rerun()

# --- 4. 結果発表フェーズ ---
elif st.session_state.phase == "result":
    st.subheader("4. 投票結果 ＆ 勝敗発表")

    # 最多得票者を特定
    executed = max(st.session_state.votes, key=st.session_state.votes.get)
    executed_role = st.session_state.roles[executed]

    st.write(f"追放されたのは **{executed}** さん（役職: {executed_role}）でした！")

    if executed_role == "人狼":
        st.balloons()
        st.success("🎉 村人陣営の勝利です！人狼を追放しました。")
    else:
        st.error("🐺 人狼陣営の勝利です！村人が追放されてしまいました。")

    st.write("--- 全員の役職 ---")
    for p, r in st.session_state.roles.items():
        st.write(f"- {p}: {r}")

    if st.button("最初からもう一度遊ぶ"):
        for key in list(st.session_state.keys()):
            del st.session_state[key]
        st.rerun()

