# Treino Pélvico

Aplicativo web de uma tela só que conduz uma sessão de exercícios do assoalho pélvico,
com contagem visual, bipes e voz em português. Roda em qualquer navegador, sem servidor,
sem instalação e sem depender da síntese de voz do aparelho.

Publicado em: <https://leoxuxu.github.io/treino-pelvico>

---

## 1. O protocolo

A sessão segue uma prescrição individual de fisioterapia pélvica, no formato do
esquema PERFECT (Laycock & Jerwood, 2001). São duas partes, feitas 2× ao dia.

| | Parte 1 — resistência | Parte 2 — força a 60% |
|---|---|---|
| Contração | 2 s, força máxima | 5 s, a 60% da força |
| Relaxamento | 4 s | 4 s |
| Repetições | 10 | 5 |
| Blocos | 3 ciclos | 3 movimentos |
| Entre blocos | 1 min | 1 min |
| Movimentos | — | afastar as pernas · empurrar o chão · ponte (só calcanhares) |

Durações: **completo 10:36**, só parte 1 **5:03**, só parte 2 **4:36**.
O áudio-guia gravado cobre tudo em **11:38** (é mais longo porque inclui as explicações faladas).

Antes de cada bloco da parte 2 há uma fase curta de **Posição** (5 s, ou 8 s na entrada
da parte 2), que anuncia o movimento e dá tempo de se ajustar.

---

## 2. Arquivos

| Arquivo | Papel |
|---|---|
| `pelvico.html` | **a fonte do app** — HTML, CSS e JS numa página só (370 linhas) |
| `clipes.js` | banco de voz: 23 clipes AAC em base64 (359 KB). **Gerado**, não editar à mão |
| `build_clipes.py` | grava os clipes com a voz do macOS e escreve o `clipes.js` |
| `build_site.py` | monta as versões publicáveis a partir de `pelvico.html` + `clipes.js` |
| `build_audio.py` | monta o áudio-guia completo (fala + bipes) e exporta em AAC |
| `Guia-Exercicios-Pelvicos.m4a` | áudio-guia completo, 11:38 (2,1 MB) |
| `publicar/` | versão em dois arquivos: `index.html` (378 KB) + `guia.mp4` |
| `publicar-1arquivo/index.html` | **o que vai para o GitHub Pages** — tudo embutido (3,2 MB) |
| `Treino-Pelvico.html` | idêntico ao anterior; cópia para abrir direto no Mac, offline |
| `.claude/launch.json` | configuração do servidor local de teste (`python3 -m http.server 8777`) |

Os três scripts são independentes e podem ser rodados em qualquer ordem; só o `build_site.py`
precisa que os outros já tenham rodado, porque consome o que eles produzem.

---

## 3. Como o app funciona

### 3.1 O plano da sessão

Tudo parte de uma lista de **blocos** (`B1` para a parte 1, `B2` para a parte 2). Cada bloco
declara `{parte, reps, hold, rest, mov, clipe, setup}`.

A função `montar(modo)` expande esses blocos num **plano linear de passos**, onde cada passo é
`{k, d, b, r}` — tipo, duração em segundos, índice do bloco e número da repetição:

```
prep(3s) → [setup] → hold, rest, hold, rest… → pause(60s) → [setup] → …
```

Os tipos de passo (`k`) são: `prep`, `setup`, `hold`, `rest`, `pause`.
`montar` também pré-calcula `starts[]` (instante de início de cada passo), `TOTAL` e `binfo[]`
(início e duração de cada bloco, usados nas barras de progresso), e redesenha as barras,
as bolinhas de repetição e o texto de ajuda.

Os três modos — `tudo`, `p1`, `p2` — são só três maneiras de montar a mesma lista de blocos.

### 3.2 O relógio

`tick(ts)` roda em `requestAnimationFrame` e acumula o tempo real decorrido em `elapsed`.
A cada quadro ele:

1. descobre em que passo está (comparando `elapsed` com `starts[]`) e, se mudou, chama `enter(i)`;
2. atualiza o anel (`stroke-dashoffset`), o número grande, o tempo restante e as barras;
3. dispara avisos falados por limiar, via `fire(chave, condição, ação)` — um `Set` (`marks`)
   garante que cada aviso toque uma vez só dentro do passo.

`enter(i)` é o ponto único onde a fase muda: define cor e rótulo, escreve o subtítulo,
toca o bipe, dispara o clipe de voz e atualiza as bolinhas.

Por que `requestAnimationFrame` e não `setInterval`: o tempo vem de `elapsed += (ts - last)/1000`,
então o relógio não acumula desvio. **Consequência importante**: `rAF` congela quando a aba vai
para segundo plano, então a sessão pausa se o usuário trocar de app (veja Limitações).

### 3.3 Áudio

Dois caminhos, ambos pelo **Web Audio API**, no mesmo `AudioContext`:

- **Bipes** — `tone(freq, dur, vol)` cria um oscilador senoidal com envelope.
  Ganhos atuais: 0,58 contrai · 0,46 solta · 0,52 fim de bloco · 0,48 posição · 0,36 contagem · 0,60 final.
- **Voz** — 23 clipes AAC embutidos em base64 em `window.CLIPS`. No primeiro toque,
  `loadClips()` converte cada um para `ArrayBuffer` e chama `decodeAudioData`;
  `say(chave, texto)` toca o `AudioBuffer` por um `AudioBufferSourceNode`.

**Por que clipes gravados e não `speechSynthesis`:** num iPhone 17 Pro a síntese do sistema
simplesmente não emitia som (os bipes funcionavam), e a voz do iOS ignorava o acento de
"Sólta", falando "sôlta". Com clipes, a pronúncia é sempre a mesma e o áudio usa o mesmo
caminho dos bipes, que funciona em todo lugar. A síntese do aparelho continua como reserva:
o segundo argumento de `say()` é o texto falado se o clipe não estiver disponível.

**A pronúncia de "Sólta":** sem o acento agudo, a voz Luciana pronuncia a vogal fechada.
Verificado por análise de formantes: "Solta" e "Sôlta" geram áudio idêntico (F ≈ 196 Hz),
enquanto "Sólta" sobe para ≈ 1016 Hz, igual ao "Só" isolado. Vale também para "sólte".

**Regras de navegador que o código respeita:**
- som só começa depois de um gesto do usuário — por isso tudo parte do botão Iniciar;
- no iOS, a primeira fala sintetizada precisa sair de dentro do gesto (`warmTTS()`);
- o interruptor de silencioso do iPhone corta o áudio no Safari (não há contorno).

### 3.4 Tela

Layout em coluna com `height:100%`, sem rolagem: cabeçalho, seletor de modo, barras de bloco,
o anel (único elemento flexível), bolinhas de repetição, tempos, botões e o player opcional.

Cores por fase via a variável `--phase`, trocada em `setPhase()`:
rosa (contrai), **âmbar** (contrai a 60%), verde-azulado (solta), azul (descanso e posição).
Tudo em tokens CSS com variantes para tema claro e escuro.

`navigator.wakeLock` mantém a tela acesa durante a sessão, e é liberado ao pausar ou terminar.

---

## 4. Build

### 4.1 Voz — `build_clipes.py`

```bash
python3 build_clipes.py
```

Para cada uma das 23 falas: `say -v Luciana -r 185` → WAV 22,05 kHz mono → corte do silêncio
(limiar 250, margem de 20 ms) → `afconvert` para AAC 40 kbps → base64 → `clipes.js`,
que define `window.CLIPS`.

Trocar de voz é mudar a constante `VOICE` no topo do arquivo. `say -v '?'` lista as instaladas.
Depois de regerar os clipes, rode sempre o `build_site.py`.

### 4.2 Áudio-guia — `build_audio.py`

```bash
python3 build_audio.py
```

Gera o `Guia-Exercicios-Pelvicos.m4a` inteiro: a introdução falada, os três ciclos da parte 1,
os três blocos da parte 2, os avisos de descanso e o encerramento, com os bipes nas mesmas
posições em que o app os toca. É o mesmo conteúdo da sessão, para quem prefere seguir só de ouvido.

### 4.3 Páginas — `build_site.py`

```bash
python3 build_site.py
```

Lê `pelvico.html` e `clipes.js` e produz três saídas:

| Saída | O que muda |
|---|---|
| `publicar/index.html` | clipes embutidos; o áudio-guia fica fora, como `guia.mp4` |
| `publicar-1arquivo/index.html` | clipes **e** áudio-guia embutidos (`data:` URI) |
| `Treino-Pelvico.html` | cópia da anterior, para abrir local |

As três recebem um `<!doctype html>` e um reset mínimo, porque `pelvico.html` é escrito sem
esse invólucro (é também a fonte publicada como artifact do Claude, que adiciona o seu próprio).

### 4.4 Teste local

```bash
python3 -m http.server 8777
```

E abra `http://localhost:8777/publicar-1arquivo/index.html`. O arquivo `.claude/launch.json`
já traz essa configuração para o painel de pré-visualização.

---

## 5. Publicação

**GitHub Pages** (`leoxuxu/treino-pelvico`): envie `publicar-1arquivo/index.html` como
`index.html` na raiz do repositório — Add file → Upload files → Commit changes.
O deploy leva de 20 a 60 segundos.

**Cuidado com cache.** O GitHub serve a página com `cache-control: max-age=600`, e o Safari
segura bem mais que isso. Para conferir uma atualização, abra com um parâmetro novo
(`?v=3`) ou numa aba privada. Se o site estiver como ícone na Tela de Início, o atalho guarda
um cache próprio: apague e adicione de novo. Para verificar o que está sendo servido de fato:

```bash
curl -sS -o /dev/null -w '%{size_download}\n' "https://leoxuxu.github.io/treino-pelvico/?cb=$RANDOM"
```

**Artifact do Claude** (privado): `pelvico.html` publicado com `clipes.js` como arquivo anexo.

**O que não funciona:** Netlify Drop sem conta — o Netlify protege com senha todo site anônimo
não reivindicado (401). Google Drive, iCloud e Dropbox entregam o HTML como download, não como página.

---

## 6. Receitas

**Mudar tempos ou repetições** — em `pelvico.html`, nas constantes `B1`/`B2`
(`hold`, `rest`, `reps`, `setup`) e em `PREP`/`REST`. O plano, as barras, as bolinhas e o
tempo total se ajustam sozinhos.

**Acrescentar um movimento à parte 2** — some um objeto a `B2` com seu `mov` e `clipe`,
acrescente a fala correspondente em `build_clipes.py` e regere clipes e site.

**Trocar a voz** — mude `VOICE` em `build_clipes.py` e rode os dois builds. Para uma voz
premium da Apple, baixe antes em Ajustes do Sistema → Acessibilidade → Conteúdo Falado.
Os arquivos da voz ficam só no Mac: no app entra apenas o áudio já renderizado.

**Mudar o volume dos bipes** — os ganhos nas chamadas `tone(...)` dentro de `enter()` e
`tick()`. Mantenha a soma com a voz abaixo de 1,0 para não distorcer.

**Regerar o áudio-guia** — `python3 build_audio.py`. Ele monta uma linha do tempo em memória
(22,05 kHz mono), posiciona cada fala e cada bipe por instante, exporta WAV e converte para AAC
48 kbps. As falas ficam em cache na pasta `vozcache/`, então a segunda execução é rápida.
O script imprime a duração e o pico — hoje 11:38 e 23.286 de 32.767, com folga antes de
distorcer. Se a duração mudar, atualize o rótulo "(11:38)" em `pelvico.html` (duas ocorrências).

Variáveis de ambiente opcionais: `VOZ_CACHE`, `SAIDA_WAV` e `SAIDA_M4A` mudam os caminhos,
úteis para gerar uma versão de teste sem sobrescrever a publicada.

---

## 7. Limitações conhecidas

- **A sessão pausa em segundo plano.** `requestAnimationFrame` congela quando a aba perde foco,
  então trocar de app interrompe a contagem. O wake lock evita o caso mais comum (tela apagando),
  mas não resolve o resto.
- **Nada é registrado.** Não há histórico, estatística ou qualquer dado salvo entre sessões.
- **Silencioso do iPhone** corta o áudio no Safari, inclusive os bipes.
- **Fontes vêm do Google Fonts**: sem internet, a página usa as fontes de sistema do aparelho.
- **3,2 MB** no arquivo único, quase tudo áudio. Carrega rápido em rede boa, mas não é leve.
- **Sem testes automatizados.** A verificação é manual, pelo navegador, instrumentando
  `AudioBufferSourceNode.prototype.start` para contar clipes e `AudioParam` para medir ganhos.
- **Um só idioma e uma só voz.**

---

## 8. Ambiente

Para rodar o app: qualquer navegador moderno. Para os builds: **macOS** (usa `say` e
`afconvert`, ambos do sistema) e **Python 3.9+**, sem dependências externas — só a biblioteca
padrão (`wave`, `array`, `base64`, `subprocess`).
