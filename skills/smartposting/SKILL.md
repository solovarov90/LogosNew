---
name: smartposting
description: "Control the SmartPosting app via its bot API: read the materials BANK, funnels, broadcasts, posts, channel, subscribers, analytics, and run actions."
metadata:
  {
    "openclaw":
      {
        "emoji": "📣",
        "requires": { "bins": ["curl"] }
      }
  }
---

# SmartPosting (app API)

Use this skill whenever Pyotr asks about the SmartPosting APP: materials BANK (банк), funnels (автоворонки), broadcasts (рассылки), posts, channel, subscribers, analytics. The full action/query CATALOG and system map are in your AGENTS.md (sections «КАРТА СИСТЕМЫ», «SMARTPOSTING — ОПЕРАТОР», «БАНК»). This skill is the TOOL to actually call that API via curl.

The environment provides `$SMARTPOSTING_API_URL` (e.g. https://smart-posting.ru) and `$SMARTPOSTING_BOT_KEY`.

## READ context / bank (query)

```bash
curl -s -X POST "$SMARTPOSTING_API_URL/api/bot/query" \
  -H "Authorization: Bearer $SMARTPOSTING_BOT_KEY" -H "Content-Type: application/json" \
  -d '{"action":"listBankAssets","args":{}}'
```

Response: `{"ok":true,"data":{"total":N,"byCategory":{stats,review,result,other},"byMediaType":{photo,video,document},"count":M,"assets":[{id,url,mediaType,category,tags,summary,extractedData}]}}`. THIS is the real app bank (NOT the vault file `каталог.md`). ВАЖНО: для вопросов «сколько файлов / разбивка» бери ГОТОВЫЕ `total` и `byCategory`/`byMediaType` — НЕ пересчитывай массив `assets` сам (он урезан до `count`, посчитаешь неверно). `assets` — это выборка для подбора материала. Чтобы собрать альбом конкретной категории — запрашивай `listBankAssets {"category":"stats","limit":8}`. Other queries: `listFunnels`, `funnelStats {funnelId}`, `subscribersCount`, `listBroadcasts`, `listPosts`, `postStats {postId}`.

## ACT (create a task, then execute)

```bash
RESP=$(curl -s -X POST "$SMARTPOSTING_API_URL/api/bot/tasks" \
  -H "Authorization: Bearer $SMARTPOSTING_BOT_KEY" -H "Content-Type: application/json" \
  -d '{"type":"mixed","title":"Рассылка","plan":[{"order":1,"action":"draft_broadcast","params":{"title":"...","text":"...","bankAssetId":"<id from listBankAssets>"}}]}')
echo "$RESP"
# take the task _id from the response, then (for sending actions) execute it:
curl -s -X POST "$SMARTPOSTING_API_URL/api/bot/tasks/<TASK_ID>/execute" \
  -H "Authorization: Bearer $SMARTPOSTING_BOT_KEY"
```

Actions (params are in AGENTS.md): `draft_broadcast`, `send_broadcast`, `schedule_broadcast`, `create_post`, `post_to_channel`, `create_funnel`, `add_funnel_step`, `update_funnel_step`, `run_funnel`, `update_funnel`, `save_to_bank`, `generate_image`.

## RULES

- To "see the bank / what materials there are" → ALWAYS call `listBankAssets`. NEVER use the vault file `Тезисы — где какой материал применять.md` for that (it is your text notes, different content).
- When composing ANY broadcast / post / funnel step → first `listBankAssets`, pick a fitting proof (by summary/category/tags) and attach it via `bankAssetId`.
- Confirm `ref`/result from the API response; do not claim success without it.

## MEDIA RULES (вложения из банка)

- Несколько фото из банка → передавай массив `bankAssetIds: ["id1","id2",...]` (НЕ один bankAssetId). Они уйдут АЛЬБОМОМ (Telegram media group, до 10).
- ОТЗЫВЫ и СТАТИСТИКА — почти всегда лучше слать ПАЧКОЙ: бери 6-8 скринов одной категории → один альбом (bankAssetIds). Один отзыв-скрин выглядит слабее, чем пачка.
- ВИДЕО — всегда ОДИНОЧНО: один bankAssetId видео (в альбом с фото не кладётся). Если в bankAssetIds попадёт видео — система отправит только его.
- МОЁ ФОТО (Пётр) или ОБЛОЖКА ГАЙДА — одиночное фото: один bankAssetId.
- Эти же правила для шага воронки (add_funnel_step / update_funnel_step) и поста.

## ОБЗОР БАНКА — показывай ОБЕ оси

При вопросе «сколько / что в банке / разбивка» выводи ДВЕ группировки (оба поля приходят с сервера):
- ПО ТИПУ (byMediaType): фото, видео, гайды (=document). ВИДЕО и ГАЙДЫ видны ТОЛЬКО здесь — обязательно назови их числа.
- ПО СМЫСЛУ (byCategory): статистика(stats), отзывы(review), результаты(result), прочее(other).
Видео и гайды свёрнуты внутрь категорий (видео обычно в result, гайды-PDF в other) — отдельной строкой их видно только в разбивке ПО ТИПУ. Никогда не теряй видео и гайды в ответе.

## СБОРКА ВОРОНКИ ПО ШАГАМ — формат add_funnel_step

funnelId обязателен. Параметры ПЛОСКИЕ (если вложишь в "step" — тоже поймём). Добавляй ПО ОДНОМУ, после каждого вызывай listFunnels и проверяй, что счётчик шагов вырос.
- сообщение: add_funnel_step {"funnelId":"<id>","type":"message","content":"<HTML текст>","buttons":[{"text":"Перейти в канал","url":"https://t.me/solovarovp"}]}
- альбом из банка: add_funnel_step {"funnelId":"<id>","type":"message","content":"<HTML>","bankAssetIds":["id1","id2","..."]}
- задержка (24ч): add_funnel_step {"funnelId":"<id>","type":"delay","delayMinutes":1440}  (480=до вечера, 960=до утра)
- гейт подписки: add_funnel_step {"funnelId":"<id>","type":"gate","channelId":"@solovarovp","content":"<HTML предложение подписаться>"}
ВАЖНО: всегда передавай type и (для message) content, (для delay) delayMinutes. Пустой content = пустой шаг.

## ЧИТАЙ АКТУАЛЬНЫЙ ТЕКСТ ШАГА ИЗ funnelStats (не из памяти!)

Когда спрашивают «что в шаге / какое первое сообщение / сколько символов» — НЕ отвечай по памяти (текст мог измениться в интерфейсе). Вызови funnelStats {"funnelId":"<id>"} и читай по каждому шагу: contentPreview (начало текста), contentLength (длина), mediaCount (сколько медиа), buttons, delayMinutes. Заготовки вроде «[Манифест...]» Пётр заменяет в UI — твоя память устаревает, источник истины = funnelStats.

## ВЫБОР ПРУФОВ ИЗ БАНКА — бери СИЛЬНЕЙШИЕ и по теме

listBankAssets отдаёт ассеты по дате (новые первыми) — НЕ хватай просто первые N. Алгоритм выбора для пруф-альбома:
1. Если Пётр отметил сильные ассеты тегом «топ» — бери ИХ: listBankAssets {"category":"stats","tag":"топ"} (или review/result). Это его отобранные флагманы.
2. Если тега «топ» нет — прочитай summary ВСЕХ ассетов категории и выбери 6-8 с САМЫМИ БОЛЬШИМИ цифрами (млн просмотров, тысячи подписчиков) и под тезис сообщения. Большие числа = сильный пруф.
3. Для шага про ОХВАТЫ — бери скрины про просмотры/охваты/рост; про ОТЗЫВЫ — review; про ДЕНЬГИ — result (оплаты/регистрации). Не мешай категории невпопад.
Перед вставкой назови Петру, какие именно ассеты берёшь и почему (по summary), чтобы он мог поправить.
