---
name: reel-ccb
description: Edita Reel vertical 9:16 de talking head (uma pessoa falando pra câmera). A Fase 1 (corte, J-cut, nível de voz, cor, verificação e preview de aprovação) é delegada à skill motor-reel; a Fase 2 é a direção — hook escrito, texto atrás da pessoa, split screen, prova na tela e legenda. Gatilhos "edita esse reel", "monta o reel", "corta meus erros", "edita no padrão CCB".
---

# Reel CCB

Duas fases, duas skills. A engenharia de corte e áudio vem da `motor-reel`, que
resolve com medição: bordas acústicas da fala, J-cut, nível por trecho e
verificação numérica. Esta skill cuida da DIREÇÃO, ou seja, do que aparece na tela.

## A regra de ouro

**Não reimplemente nada que a `motor-reel` já faz.** Se a dúvida é sobre corte,
nível, cor, sincronismo, encoding ou aprovação, a resposta está no SKILL.md da
motor-reel e nas references dela. Esta skill só manda na Fase 2.

## FASE 1 — delegada, sem exceção

Roda a Fase 1 da `motor-reel` como está escrito lá, com três acréscimos:

1. **Passe o idioma explícito** (`--language pt`, `en`...). A auto-detecção custa
   uma passada e erra em clipe curto.
2. **`voice_levels.py` é obrigatório.** Quase todo mundo deixa a voz cair no fim
   da frase, e nem o transcript nem o ouvido no fone pegam isso. Corrija com
   `gain_db` por range, dimensionado pelo déficit que o `verify_cut` mede, não
   pelo pior trecho isolado.
3. **Áudio de celular costuma chegar perto de −25 LUFS.** O loudnorm da Fase 1
   entrega −14. Denoise/de-reverb só quando o ambiente pedir, e sempre ANTES,
   no material bruto.

Gate de aprovação: o preview da motor-reel (`preview_server.py`), não print no chat.

## FASE 2 — a direção

Tudo é descrito em `remotion/public/edit-data.json`. O template TSX é imutável;
gráfico sob medida só em `src/CustomGraphics.tsx`.

### As 3 leis

1. **A pessoa nunca sai da tela.** Menos de 2% do tempo sem ela visível. O modo
   padrão é split, não b-roll em tela cheia.
2. **Fundo nunca é preto chapado.** Todo overlay que tira a pessoa do quadro usa
   o próprio ambiente borrado.
3. **Os 2 primeiros segundos são a pessoa falando com o hook escrito atrás.**
   Nunca abrir com b-roll, cartaz ou logo. O frame 1 é a thumb do feed.

### Técnicas → mecanismo

| técnica | onde vive |
|---|---|
| hook escrito na abertura | `hook` (`style: outline`, duas linhas, até ~6 palavras) |
| texto atrás da pessoa | `behind[]` com `kind: "words"` + `person_matte.py` (matte ProRes 4444 com alpha, já trata halo e drift) |
| split screen (rosto em cima, apoio embaixo) | `splitInserts[]` com `bandH` |
| prova na tela (print + etiqueta) | `CustomGraphics.tsx`, no instante em que a afirmação é dita |
| legenda palavra por palavra | `captions` (`karaoke`, fonte bold, caixa alta, com contorno) |
| zoom e respiração no rosto | `camera.zooms` + `camera.pushIn` |

### Ritmo

- Rosto puro não passa de 4-5s seguidos.
- Nunca dois cards estáticos em sequência.
- Ordem de preferência do apoio: footage real > demo real > b-roll de IA > logo estático.
- Toda afirmação factual verificável ganha prova na tela quando é dita.

### Gotchas da Fase 2 (medidos em edição real)

1. **`segments.json` sai do `jcut_timeline`, nunca da soma dos segmentos.** A
   soma dos `clips_graded/seg_*.mp4` erra ~234ms (o encavalamento do J-cut), e
   todo overlay entra progressivamente adiantado: renderiza limpo e só se
   percebe assistindo.
2. **O schema é `{"segments":[{"start","dur"}]}`**, não uma lista de fronteiras.
   Errar derruba o render com `Cannot read properties of undefined`.
3. **Confira as tags de cor DEPOIS do render.** O `cut.mp4` sai
   `tv/bt709/bt709/bt709`, mas o render do Remotion volta `pc/bt470bg/unknown`
   (quem renderiza é o Chrome), uns 20% mais saturado. Correção:
   ```bash
   ffmpeg -y -i final.mp4 -vf "scale=in_range=pc:out_range=tv,format=yuv420p" \
     -c:v libx264 -preset medium -crf 18 \
     -x264opts "colorprim=bt709:transfer=bt709:colormatrix=bt709:fullrange=off" \
     -color_range tv -c:a copy final_bt709.mp4
   ```
   `-color_primaries`/`-color_trc` sozinhos não gravam as tags; via `-x264opts`
   gravam. Valide comparando `signalstats` do final contra o corte.
4. **Loudnorm na Fase 2 não é opcional.** O render do Remotion já chega
   clipando (+0,40 dBTP medido) antes de qualquer trilha.
5. **Legenda `scatter` ignora `captions.windows`** e cai em cima do rosto no
   split. Use `karaoke`.
6. **`bandH` alto demais vira close extremo** quando a pessoa grava com o
   celular perto do rosto. Comece em ~620. Pra usar split como padrão, grave
   um passo mais afastado.

### Proibido

- **Tirar ponto de corte do timestamp do Whisper.** As bordas vêm do `speech_regions.py`.
- **Cortar com margem simétrica.** Entrada −30ms, saída +50 a +80ms: o fim da palavra precisa do decay.
- **Emendar take com take seco (butt join).** J-cut é o padrão.
- **Consertar take baixo com compressor global.** Levanta o ruído junto. Ganho por range.
- **Renderizar e assistir pra saber se ficou bom.** `verify_cut.py` primeiro; imagem só onde ele apontar.
- **Ler `transcripts/*.json` no contexto.** Ler `takes_packed.md`.
- **Texto queimado com ffmpeg/PIL na Fase 2.** É Remotion.

### Checklist antes de entregar

[ ] frame 1 = pessoa + hook escrito? · [ ] tempo sem a pessoa < 2%? · [ ] nenhum
fundo preto chapado? · [ ] `verify_cut` deu `clean` e níveis `balanced`? · [ ] toda
afirmação factual com prova? · [ ] `cut.mp4` e `final.mp4` marcados
tv/bt709/bt709/bt709? · [ ] pico conferido antes do loudnorm? · [ ] `segments.json`
veio do `jcut_timeline`?

## Referências

- `../motor-reel/SKILL.md` — a Fase 1 inteira e as Hard Rules.
- `../motor-reel/references/shortform.md` — o schema e os gotchas da Fase 2.
