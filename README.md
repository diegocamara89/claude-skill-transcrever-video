# transcrever-video

Skill para Claude Code que faz um agente **ler o conteúdo de um vídeo** a partir de um
link — Instagram, YouTube, TikTok, X, Facebook, Reddit — ou de um arquivo local.

Modelos de linguagem não assistem vídeo. Esta skill dá o caminho: baixar com
[`yt-dlp`](https://github.com/yt-dlp/yt-dlp) e transcrever com
[`faster-whisper`](https://github.com/SYSTRAN/faster-whisper), **tudo local**, sem chave
de API e sem enviar o áudio para serviço nenhum.

## O que ela resolve

O erro comum é ir direto ao caminho mais caro: baixar o vídeo inteiro e transcrever.
A skill impõe uma ordem do mais barato ao mais caro, e na maioria dos casos o passo 1
já responde:

1. **Metadados** — `yt-dlp` traz título e descrição sem baixar nada. Em post didático,
   a descrição costuma trazer o roteiro inteiro
2. **Legendas prontas**, se o vídeo tiver
3. **Só o áudio** (~10× menor que o vídeo) + transcrição
4. **Vídeo + quadros extraídos**, apenas quando o que importa está *na tela* e não na
   fala — tutorial de interface, por exemplo

## Instalação

Copie a pasta para o diretório de skills do Claude Code:

```
~/.claude/skills/transcrever-video/
```

Dependências, instaladas sob demanda pela própria skill:

```bash
python -m pip install --upgrade yt-dlp faster-whisper
```

`ffmpeg` precisa estar no PATH. `faster-whisper` roda em CPU via ctranslate2 — não exige
torch nem GPU.

## Exemplo

> "assiste esse vídeo e me diz se ele cobre tudo que fizemos: `<link>`"

O agente busca os metadados, decide se precisa do áudio, transcreve com marca de tempo
por segmento e responde separando **o que foi dito** do **que ele inferiu**.

## Armadilhas cobertas

- Instagram exige login, mas link com `?stkn=...` funciona sem autenticação — o parâmetro
  precisa ser preservado
- `WebFetch` não serve nessas páginas: montam o conteúdo por JavaScript e devolvem só o
  título
- Os avisos do HuggingFace na primeira execução (token ausente, symlink no Windows) são
  inofensivos
- O JSON de metadados passa de 30 KB por causa da lista de `formats` — nunca imprimir
  inteiro

## Tabela de modelos

| Modelo | Velocidade em CPU | Quando usar |
|---|---|---|
| `tiny` | muito rápida | só para saber o assunto |
| `base` | rápida | áudio limpo |
| `small` | ~1× tempo real | **padrão recomendado** |
| `medium` | lenta | áudio ruim, sotaque forte, termo técnico |

## Licença

MIT.
