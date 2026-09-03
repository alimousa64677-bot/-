WHERE id = ?
    """, (question_id,))

    result = cursor.fetchone()

    if not result:
        return

    correct = result[0]
    points = result[1]

    if selected == correct:

        state["score"] += points

        bot.answer_callback_query(
            call.id,
            "✅ إجابة صحيحة!"
        )

    else:

        bot.answer_callback_query(
            call.id,
            "❌ إجابة خاطئة"
        )

    state["question_index"] += 1

    send_question(
        call.message.chat.id,
        user_id
    )


# =========================================================
# إنهاء الاختبار
# =========================================================

def finish_quiz(chat_id, user_id):

    state = user_state.get(user_id)

    if not state:
        return

    quiz_id = state["quiz_id"]
    score = state["score"]

    cursor.execute("""
        SELECT SUM(points), COUNT(*)
        FROM questions
        WHERE quiz_id = ?
    """, (quiz_id,))

    result = cursor.fetchone()

    total_points = result[0] or 0
    question_count = result[1] or 0

    username = ""

    try:
        username = bot.get_chat(user_id).username or ""
    except Exception:
        pass

    cursor.execute("""
        INSERT INTO results
        (
            quiz_id,
            user_id,
            username,
            score,
            total
        )
        VALUES (?, ?, ?, ?, ?)
        ON CONFLICT(quiz_id, user_id)
        DO UPDATE SET
            score = excluded.score,
            total = excluded.total,
            created_at = CURRENT_TIMESTAMP
    """, (
        quiz_id,
        user_id,
        username,
        score,
        total_points
    ))

    db.commit()

    percentage = 0

    if total_points > 0:
        percentage = round(
            (score / total_points) * 100
        )

    bot.send_message(
        chat_id,
        f"""
🏁 <b>انتهى الاختبار!</b>

🎯 نتيجتك:
<b>{score} / {total_points}</b>

📊 النسبة:
<b>{percentage}%</b>

📝 عدد الأسئلة:
{question_count}

👏 أحسنت!
"""
    )

    user_state.pop(user_id, None)


# =========================================================
# الإحصائيات
# =========================================================

@bot.callback_query_handler(
    func=lambda call: call.data == "stats"
)
def stats(call):

    user_id = call.from_user.id

    cursor.execute("""
        SELECT COUNT(*)
        FROM quizzes
        WHERE creator_id = ?
    """, (user_id,))

    quizzes = cursor.fetchone()[0]

    cursor.execute("""
        SELECT COUNT(*)
        FROM results r
        JOIN quizzes q
        ON r.quiz_id = q.id
        WHERE q.creator_id = ?
    """, (user_id,))

    participants = cursor.fetchone()[0]

    bot.answer_callback_query(call.id)

    bot.send_message(
        call.message.chat.id,
        f"""
📊 <b>إحصائيات إدريس</b>

📚 الاختبارات: {quizzes}

👥 المشاركات: {participants}
"""
    )


# =========================================================
# عن إدريس
# =========================================================

@bot.callback_query_handler(
    func=lambda call: call.data == "about"
)
def about(call):

    bot.answer_callback_query(call.id)

    bot.send_message(
        call.message.chat.id,
        """
🤖 <b>إدريس</b>

منصة اختبارات تعليمية على Telegram.

📚 إنشاء الاختبارات
📝 أسئلة متعددة الخيارات
🎯 تصحيح تلقائي
📊 حساب النتائج
🏆 نظام قابل للتطوير
"""
    )


# =========================================================
# تشغيل البوت
# =========================================================

if name == "__main__":

    print("🤖 إدريس يعمل الآن...")

    bot.infinity_polling(
        skip_pending=True
    )
