---
name: transcrever-video
description: Use quando o usuario mandar um link de video (Instagram, YouTube, TikTok, X, Facebook, Reddit) e quiser que o conteudo seja lido, resumido, transcrito ou usado como fonte. Tambem quando o video estiver num arquivo local. Baixa com yt-dlp e transcreve com faster-whisper, localmente, sem depender de API. TRIGGERS (PT) - assiste esse video, ve esse video, transcreve esse video, resume esse reel, o que esse video diz, baixa esse video, esse tiktok, esse reel, esse short, legenda do video, audio do video, extrai o texto do video. TRIGGERS (EN) - watch this video, transcribe this, summarize this reel, download this video, what does this video say, get the transcript.
---

# Transcrever video a partir de link

O modelo nao assiste video. Este e o caminho: **baixar** com `yt-dlp` e **transcrever**
com `faster-whisper`, tudo local. Funciona sem chave de API.

## Ordem de ataque (do mais barato ao mais caro)

### 1. Metadados primeiro — quase sempre resolve

`yt-dlp` traz titulo, descricao e legendas **sem baixar o video**. Em posts didaticos
(Instagram, YouTube) a descricao costuma trazer o roteiro inteiro. Comece aqui:

```bash
python -m yt_dlp --skip-download --print-json "<URL>" > meta.json
```

Depois leia `description`, `title`, `uploader`. Se a descricao ja responde, pare.

Cuidado: o JSON traz a lista completa de `formats` e fica enorme (30 KB+). Extraia
so o que interessa em vez de imprimir tudo:

```bash
python -c "import json;d=json.load(open('meta.json',encoding='utf-8'));print(d['title']);print(d.get('description',''))"
```

### 2. Legendas prontas, se existirem

```bash
python -m yt_dlp --skip-download --write-auto-subs --sub-langs "pt.*,en.*" --convert-subs srt -o "sub" "<URL>"
```

Se sair um `.srt`, leia e pare. Muito mais rapido que transcrever.

### 3. Baixar so o audio e transcrever

Nunca baixe o video se o objetivo e o texto — o audio e ~10x menor.

```bash
python -m yt_dlp -q -x --audio-format mp3 -o "audio.%(ext)s" "<URL>"
```

```python
from faster_whisper import WhisperModel
m = WhisperModel("small", device="cpu", compute_type="int8")
segs, info = m.transcribe("audio.mp3", language="pt", vad_filter=True)
print(f"duracao: {info.duration:.0f}s")
for s in segs:
    print(f"[{int(s.start//60)}:{int(s.start%60):02d}] {s.text.strip()}")
```

Sempre passar `language=` quando souber o idioma: evita deteccao errada e acelera.
`vad_filter=True` corta silencio e reduz alucinacao em trechos sem fala.

### 4. Video inteiro, so se precisar do que esta NA TELA

Se o conteudo importante for texto na tela (menus, valores, cliques) e nao a fala,
baixe o video e extraia quadros para leitura visual:

```bash
python -m yt_dlp -q -o "reel.%(ext)s" "<URL>"
ffmpeg -i reel.mp4 -vf fps=1/5 -q:v 2 quadro_%03d.jpg
```

Depois use a ferramenta de leitura de imagem nos quadros relevantes. Um quadro a cada
5 segundos costuma bastar para tutorial de interface.

## Instalacao (so se faltar)

Verifique antes de instalar:

```bash
for t in ffmpeg python; do printf "%-8s " $t; command -v $t >/dev/null && echo ok || echo AUSENTE; done
python -m yt_dlp --version 2>/dev/null || echo "yt-dlp ausente"
python -c "import faster_whisper" 2>/dev/null && echo "faster-whisper ok" || echo "faster-whisper ausente"
```

```bash
python -m pip install --quiet --upgrade yt-dlp faster-whisper
```

`ffmpeg` e obrigatorio para extrair audio e quadros. `faster-whisper` nao precisa de
torch nem GPU — usa ctranslate2 e roda em CPU.

## Escolha do modelo

| Modelo | Velocidade em CPU | Quando usar |
|---|---|---|
| `tiny` | muito rapida | so para saber o assunto |
| `base` | rapida | audio limpo, fala clara |
| `small` | ~1x tempo real | **padrao recomendado** |
| `medium` | lenta | audio ruim, sotaque forte, termo tecnico |

Para video de 2 a 3 minutos, `small` em `int8` leva pouco mais que a duracao do audio.

## Armadilhas verificadas

**Instagram exige login** para acesso normal, mas link com `?stkn=...` (token de
compartilhamento) funciona sem autenticacao — preserve o parametro na URL. Sem ele,
espere erro de login.

**`WebFetch` nao serve** para essas paginas: elas montam o conteudo por JavaScript e
voltam so o titulo "Instagram". Nao insista; va direto ao `yt-dlp`.

**Avisos do HuggingFace na primeira execucao sao inofensivos:**
- `unauthenticated requests to the HF Hub` — so limita taxa de download
- `cache-system uses symlinks... machine does not support them` — no Windows sem Modo
  Desenvolvedor, o cache duplica arquivo em vez de linkar. Funciona igual, gasta mais disco.

**O primeiro `transcribe` baixa o modelo** (centenas de MB). Se parecer travado, e
download. Rodadas seguintes usam o cache.

**Nunca imprimir o JSON de metadados inteiro** — a lista de `formats` estoura o limite
de saida e desperdica contexto.

## Boas praticas de entrega

- Transcricao com **marca de tempo** por segmento: o usuario consegue conferir na fonte.
- Ao resumir, **separe o que foi dito do que voce inferiu.** Transcricao de audio erra
  numero e nome proprio — no material tecnico, sinalize valores como "transcrito, conferir".
- Guarde os arquivos no diretorio temporario da sessao, nao na pasta do usuario.
- Se o video for tutorial de interface, combine: transcricao para a narracao **e**
  quadros para os valores na tela.
