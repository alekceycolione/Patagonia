# Sonificação · Hero da Patagônia

Mapeamento sonoro da sequência de 240 frames. Cada frame tem:

- **`wind_db` / `drone_db` / `sub_db`** — amplitude em dB (mais perto de 0 = mais alto, `-inf` = mudo)
- **`shimmer_n`** — grãos de alta-frequência por segundo (8-12 kHz)
- **`pan`** — posição no campo stereo (-1 a 1)
- **`mood`** — vibe poética (para briefing de sound designer)
- **`texture`** — material tátil evocado (areia, musgo, gelo, estepe, cristal)

## Como usar

### 1. Sound design manual
Abrir no editor de áudio e usar `mood` + `texture` como brief. Os 7 `phases` dão a macro-forma.

### 2. Geração automatizada
- **Reaper + ReaSynth:** importar o JSON, usar `wind_db` como volume de uma track de noise
- **Ableton + Max4Live:** mapear `pan` para automação de stereo
- **Pure Data / Max/MSP:** ler o JSON em array, aplicar envelopes

### 3. Web (Web Audio API)
```js
const data = await fetch('src/sound/patagonia-sonification.json').then(r => r.json());

// No onUpdate da timeline:
const f = data.frames[Math.floor(heroTL.progress() * 239)];
windGain.gain.value = dbToLinear(f.wind_db);
droneGain.gain.value = dbToLinear(f.drone_db);
stereoPanner.pan.value = f.pan;
```

## Macro-forma

```
[1────30]    silêncio_antes_do_vento    areia_seca_ao_fundo
[31───60]    brisa_visível              musgo_e_cristais
[61───90]    drone_puxa_terra           estepe_e_gelo         ← SUB ENTRA
[91──120]    drift_principal            cristais_empilhados   ← SHIMMER PICO
[121──160]   movimento_continuo         estepe_e_cristais
[161──200]   esmaecimento_longo         estepe_vazia          ← SUB SAI
[201──240]   retorno_ao_silêncio        areia_seca_ao_fundo
```

## Notas

- Tonalidade: F# menor (raiz 92.50 Hz)
- Sem percussão, sem melodia — vastidão
- Reverb: catedral grande vazia (decay 8.5s, wet 0.72)
- Loudness alvo: -23 LUFS (broadcast padrão)
- True peak: -1.5 dB

Inspiração: trilha de "There Will Be Blood" (Jonny Greenwood) + rookery recordings + drone metal (Sun O)))).
