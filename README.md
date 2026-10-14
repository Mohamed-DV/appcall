# Sofia Fast ⚡ — أجوبة فورية + "تدريب" بـ OpenAI / Claude

## الفكرة فـ سطرين

- **وقت التدريب (مرة وحدة، offline):** OpenAI أو Claude كيلعب دور **الأستاذ** → كيكتب آلاف الطرق اللي كيسول بيها الكليان كل سؤال (بأخطاء Vosk، بالدارجة الفرنسية ديال إفريقيا...)، و موديل ثاني كيلعب دور **الحكم (juge)** كيتأكد بلي كل جواب صحيح حسب fiche produit.
- **وقت المكالمة:** Sofia ما كتعيط لحتى API. كتعرف **النية** (prix، posologie، livraison à Douala...) فـ **~0.2 ms** و كتقرا الجواب المُتحقق منه. إلا جا سؤال جديد → LLM بالـ streaming، و السؤال كيتسجل و كيتعلّم.

```
Client (Vosk) ──► 0. garde-fous santé (regex, 0.05 ms)   "j'arrête mon insuline ?"
                  1. routeur d'intentions (0.2–3 ms)      ← kb.json (آلاف الصيغ)
                  2. LLM streaming phrase par phrase      ← OpenAI → Claude → Ollama
                     + contrôle de chaque phrase avant TTS
                  3. سؤال جديد → kb_learned.json (فوري) + misses.jsonl → `learn`
```

النتيجة المقاسة على 70 سؤال "مخربق" بحال Vosk، **غير بالـ seed اللي كتبت باليد (312 جملة، بلا API):**
91% صحيح، **0.16 ms** فـ المتوسط. بعد `build` بالأستاذ غادي يطلع أكثر.

---

## الملفات

| ملف | الدور |
|---|---|
| `fast_agent.py` | العقل وقت المكالمة (كيعوّض `DualAgent` / `OrderAgent`) |
| `intent_router.py` | التعرّف على النية: exact → lexical (n-grams) → embeddings (اختياري) |
| `sofia_train.py` | التدريب: `build`, `learn`, `eval`, `review`, `export-ft` |
| `llm_providers.py` | OpenAI / Claude / Ollama / fake بنفس الواجهة |
| `sofia_common.py` | config + placeholders + سجل الأسئلة الجديدة |
| `sofia_live.py` | التشغيل: ميكروفون أو `--text` |
| `config.glucocontrol.json` | config ديال GlucoControl (مبني على config ديالك) |
| `products/glucocontrol/product.json` | الحقائق من صفحة nutrafrik.net |
| `products/glucocontrol/kb_seed.json` | 67 نية + أجوبة مكتوبة باليد و مُتحقق منها |

---

## 0. التثبيت

```bash
pip install -r requirements.txt
export OPENAI_API_KEY="sk-..."
export ANTHROPIC_API_KEY="sk-ant-..."
export SOFIA_CONFIG=config.glucocontrol.json
```

⚠️ المفاتيح ديما فـ variables d'environnement، عمرها فـ config.json ولا فـ الكود.

## 1. جرّب دابا (بلا API، بلا ميكرو)

```bash
python sofia_live.py --text --no-llm
```

## 2. "التدريب" بالأستاذ

```bash
# تجربة مجانية للـ pipeline كامل (fake teacher)
python sofia_train.py build --teacher fake --judge none --rounds 2 --yes

# التدريب الحقيقي: Claude أستاذ، OpenAI حكم
python sofia_train.py build --teacher anthropic --judge openai --rounds 4
```

- `--rounds` = شحال من دفعة صيغ لكل نية (كل دفعة بأسلوب مختلف: قصير، Vosk مخربق، فرنسية إفريقية، كليان شاك، شخص كبير...).
- عدد الطلبات ≈ `نيات × rounds` + الحكم. مع 67 نية: `--rounds 4` ≈ 300 طلب و **~5000 صيغة**. بغيتي ~1000 طلب؟ `--rounds 14`. من بعد 5–6 الربح كيولي صغير.
- السكريبت كيقولك عدد الطلبات قبل ما يبدا، و كيدير **cache** (`.teacher_cache/`): إلا توقف فـ النص، عاود نفس الأمر و ما تخلصش مرتين.
- `--gaps 6`: الأستاذ كيقترح نيات ناقصة فـ كل catégorie، و الحكم كيتحقق منها.

## 3. القياس + معايرة العتبات

```bash
python sofia_train.py eval --write
```
كيعطيك Top-1، نسبة الأجوبة الفورية و دقتها، latence، و أكثر النيات اللي كيتخلطو. `--write` كيحفظ العتبات اللي كتعطي دقة ≥ 97%.

## 4. المراجعة البشرية (مهمة بزاف — منتوج صحي)

`build` كيخرج `products/glucocontrol/kb_review.csv` (كيتحل فـ Excel). قرا الأجوبة، عمّر `nouvelle_reponse` ولا `valider` (oui/non)، و من بعد:
```bash
python sofia_train.py review --import products/glucocontrol/kb_review.csv
```
النيات `verified: false` ما كتجاوبش فورياً، كتمشي للـ LLM.

## 5. بالميكروفون

```bash
python sofia_live.py
```
كل جواب كيبيّن: `[instant/lex] 1re phrase 0.3 ms · prix` ولا `[llm/openai:gpt-5.4-mini] 1re phrase 650 ms`.

## 6. التعلّم من المكالمات (كل ليلة مثلاً)

```bash
python sofia_train.py learn
```
كياخذ `misses.jsonl` → الأستاذ كيقول واش كل سؤال هو نية موجودة (كيتزاد كصيغة جديدة، الجواب ما كيتبدلش = آمن) ولا نية جديدة (جواب + حكم + صيغ). `kb_learned.json` (اللي تعلّم مباشرة فـ المكالمة) كيتصفّى.
`support_queue.jsonl` = أسئلة المعلومة ديالها ماكايناش فـ الـ fiche (made in، promo...) → زيدها فـ `product.json`.

## 7. ربطه مع `confirmation_call_agent.py`

```python
# زيد تحت السطر 26 (خلّي import OrderAgent، الـ type hints محتاجينو)
from fast_agent import FastAgent
# السطر 29
CONFIG_PATH = os.environ.get("SOFIA_CONFIG", "config.json")
# السطر 328
qa_agent = FastAgent(mode="flow")
```
`mode="flow"` = ما كيزيدش "Est-ce que vous confirmez ?" حيت السكريبت ديالك هو اللي كيسول.

## 8. Embeddings (اختياري، للأسئلة البعيدة)

```bash
ollama pull bge-m3
```
فـ config: `"semantic": {"enabled": true, "model": "bge-m3"}` ثم `python sofia_train.py eval --write`.
كيتحسب غير إلا الـ lexical ما كانش متأكد (+15–50 ms على M1).

## 9. Fine-tuning محلي (اختياري)

```bash
python sofia_train.py export-ft --out finetune_data
pip install mlx-lm
mlx_lm.lora --model Qwen/Qwen2.5-1.5B-Instruct --train --data finetune_data --iters 600
mlx_lm.fuse --model Qwen/Qwen2.5-1.5B-Instruct --adapter-path adapters --save-path sofia-fused
# GGUF ثم Ollama: convert_hf_to_gguf.py (llama.cpp) ثم Modelfile: FROM ./sofia.gguf
```
تأكد من الخيارات بـ `--help` حيت كيتبدلو بين الإصدارات. بصراحة: الـ fine-tune كيحسّن الأسلوب ديال Qwen 1.5B، ولكن **الدقة كتجي من الراوتر + الأجوبة المُتحقق منها**، ماشي من الـ fine-tune.

---

## ملاحظات

- **الثمن:** `order.price` فـ config هو اللي كيتقال (450 000 GNF ديال Conakry). بدّلو حسب بلاد الكليان.
- **الأمان:** GlucoControl كيتباع لناس عندهم السكر. السؤال "واش نحبس الأنسولين؟" عمرو ما كيوصل للـ LLM، و أي جملة من الـ LLM فيها "arrêtez votre traitement" ولا "guérit le diabète" كتتحبس قبل ما تتقال.
- **"Vous êtes un robot ?":** Sofia كتقول الحقيقة و كتقترح conseiller. القاعدة القديمة "ما تقولش بلي نتي IA" بدلتها: الكذب على الكليان مخاطرة (شكايات، و قوانين كتفرض الإفصاح فـ بزاف ديال البلدان).
- **الموديلات:** `gpt-5.4-mini`، `gpt-5.5`، `claude-haiku-5-5`، `claude-sonnet-5-5` مكتوبين فـ config — بدّلهم إلا بغيتي. إلا رفض موديل شي paramètre (temperature، reasoning_effort، thinking)، الكود كيحيدو و كيعاود بوحدو.
# appcall
