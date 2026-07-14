---
titulo: "Snake Duel — Arquitetura Técnica (Modo 1)"
data: 2026-07-14
status: "Design fechado, pronto para implementação"
---

# Snake Duel — Arquitetura Técnica

> Documento de referência para as sessões de desenvolvimento. Carregar este contexto no início de cada sessão de trabalho no projeto.

## Visão geral do projeto

**Engine:** Unity (2D)
**Rede:** Mirror Networking (host-and-join direto por IP, sem servidor dedicado)
**Modo alvo do build:** Modo 1 (Tempo Limite) — 2-3 dias
**Modo 2:** documentado separadamente, não implementado nesta fase

---

## Princípio arquitetural central

Igual fizemos no protótipo de Snake em C# (Core desacoplado de I/O), aqui a regra é:

> **A lógica de jogo (grid, movimento, colisão, itens, pontuação) não deve depender diretamente do Mirror.**
> O Mirror só entra na camada de sincronização — ele replica *estado* que a lógica pura já calculou.

Isso significa duas camadas bem separadas:

```
SnakeDuel/
├── Assets/
│   ├── Scripts/
│   │   ├── Core/                  <- lógica pura, testável, sem NetworkBehaviour
│   │   │   ├── GridPosition.cs
│   │   │   ├── SnakeState.cs
│   │   │   ├── BoardEngine.cs
│   │   │   ├── ItemSpawner.cs
│   │   │   ├── BombSystem.cs
│   │   │   └── ScoreCalculator.cs
│   │   │
│   │   ├── Network/                <- camada Mirror, "casca" fina em cima do Core
│   │   │   ├── PlayerNetwork.cs         (NetworkBehaviour do jogador)
│   │   │   ├── BoardNetworkSync.cs      (sincroniza estado do BoardEngine)
│   │   │   ├── ItemNetworkSync.cs
│   │   │   └── MatchNetworkManager.cs   (custom NetworkManager)
│   │   │
│   │   ├── UI/
│   │   │   ├── HUDController.cs
│   │   │   ├── SkillOrbPromptUI.cs
│   │   │   └── EndScreenUI.cs
│   │   │
│   │   └── Tests/
│   │       ├── BoardEngineTests.cs
│   │       ├── BombSystemTests.cs
│   │       └── ScoreCalculatorTests.cs
│   │
│   ├── Sprites/                     <- suas ilustrações (cobra, itens, tiles)
│   ├── Prefabs/
│   └── Scenes/
│       ├── MainMenu.unity           (Host / Conectar por IP)
│       └── Match.unity
```

**Por que separar assim:** você consegue escrever testes unitários pro `BoardEngine`, `BombSystem` e `ScoreCalculator` sem precisar rodar o Mirror ou sequer abrir duas instâncias do jogo — exatamente o hábito de teste que identificamos como seu maior gap no roadmap júnior.

### Assembly Definitions — tornando a separação obrigatória, não só uma convenção

Pastas sozinhas não impedem que um script do `Core` referencie `Mirror` ou `UnityEngine.Networking` por engano. Pra isso ser uma regra que o **compilador** garante (não só disciplina pessoal), cada pasta principal precisa do seu próprio `.asmdef`:

```
Scripts/
├── Core/
│   └── SnakeDuel.Core.asmdef        <- SEM referência a Mirror. Referência mínima/nenhuma a UnityEngine.
├── Network/
│   └── SnakeDuel.Network.asmdef     <- referencia Core.asmdef + Mirror
├── UI/
│   └── SnakeDuel.UI.asmdef          <- referencia Core.asmdef + Network.asmdef
└── Tests/
    └── SnakeDuel.Tests.asmdef       <- referencia Core.asmdef + com "Test Assemblies" marcado, pra rodar no Test Runner (janela Window → General → Test Runner)
```

Se em algum momento o `Core.asmdef` "pedir" pra adicionar Mirror como dependência pra compilar, é o sinal de que uma lógica que deveria estar em `Network/` vazou pra dentro do Core — trate isso como um erro de arquitetura a corrigir, não como "só adicionar a referência que falta".

---

## Camada Core (lógica pura)

### `GridPosition.cs`
Struct simples de coordenada `(x, y)` com wraparound embutido:

```csharp
public readonly struct GridPosition
{
    public int X { get; }
    public int Y { get; }
    private const int BoardSize = 24;

    public GridPosition(int x, int y)
    {
        X = ((x % BoardSize) + BoardSize) % BoardSize; // wraparound garantido
        Y = ((y % BoardSize) + BoardSize) % BoardSize;
    }
}
```

### `SnakeState.cs`
Representa o estado de UMA cobra (sem saber se é local ou remota):
- `List<GridPosition> Segments`
- `GridPosition Direction`
- `int Score`
- `bool IsAlive`
- Métodos: `Move()`, `Grow(int amount)`, `ResetToStart()`, `ApplyScorePenalty(float percent)`

### `BoardEngine.cs`
Orquestra UM tabuleiro (uma cobra + seus itens):
- Recebe tick de movimento (chamado pelo game loop no intervalo calibrado — 180ms decrescendo)
- Detecta colisão com o próprio corpo
- Detecta pickup de item na posição da cabeça
- Expõe eventos: `OnFoodEaten`, `OnSkillOrbPicked`, `OnBombPicked`, `OnSnakeDied`

### `ItemSpawner.cs`
Gerencia spawn de comida comum, maçã bufada e orbe de habilidade:
- Comida comum: sempre 1 no tabuleiro, respawna imediatamente
- Maçã bufada: 1 no tabuleiro, respawna 15s após ser comida
- Orbe de habilidade: spawn a cada ~20s

### `BombSystem.cs`
A parte mais "stateful" do Core — gerencia o ciclo de vida de bombas jogadas:
- `ThrowBomb(GridPosition target, BombType type)` → registra bomba com fuse de 3s
- `Tick(float deltaTime)` → decrementa fuse; ao chegar a 0, calcula explosão (área 3x3) e verifica se a cabeça de alguma cobra está na área
- Aplica penalidade: bomba comum -20%, mega bomba -35%, sempre + reset de posição

### `ScoreCalculator.cs`
Centraliza os valores (x10) num único lugar — evita "números mágicos" espalhados pelo código:
```csharp
public static class ScoreValues
{
    public const int ComidaComum = 10;
    public const int MacaBufada = 30;
}
```

### `SnakeAI.cs` (Modo Solo)
Um novo "cérebro" que decide direção a cada tick — plugado no mesmo `BoardEngine` que já existe, no lugar do `PlayerNetwork`. Isso funciona porque o Core nunca soube (nem precisa saber) de onde vem a direção.

**Por que o A* funciona de graça em cima do wraparound:** `GridPosition` já resolve a soma módulo internamente — então gerar vizinhos pra busca (`up/down/left/right` a partir de uma célula) já é toroidal automaticamente, sem código extra.

```csharp
public class SnakeAI
{
    public Direction DecideNextMove(BoardSnapshot board, SnakeState self)
    {
        var target = ChooseTarget(board, self);          // heurística: valor/distância
        var path = AStarPathfinder.FindPath(
            start: self.Segments[0],
            goal: target,
            isBlocked: pos => IsBlocked(pos, board, self)  // corpo próprio + área de bombas armadas
        );
        return path.Count > 1 ? DirectionBetween(path[0], path[1]) : self.Direction; // fallback: mantém direção
    }

    private GridPosition ChooseTarget(BoardSnapshot board, SnakeState self)
    {
        // prioridade = valor do item / distância até ele (Manhattan, ciente do wraparound)
        // ignora itens de habilidade/bomba como alvo direto (não fazem parte da busca de comida)
    }

    private bool IsBlocked(GridPosition pos, BoardSnapshot board, SnakeState self)
    {
        // bloqueado se: está no corpo da própria cobra
        // OU está dentro do raio 3x3 de uma bomba com fuse ativo (tratado como obstáculo temporário)
    }
}
```

**Decisões simplificadas (não são pathfinding, são regras condicionais baratas):**
- Orbe de habilidade: se `Score < ScoreDoAdversario` → escolhe debuff; senão → escolhe buff.
- Bomba: joga perto da posição atual do jogador (sem prever rota futura — fica como possível melhoria pós-prazo).

**Replanejamento:** A* roda a cada tick (grid pequeno, 24x24 = 576 células — custo computacional irrelevante mesmo recalculando toda vez que o tabuleiro muda).

---

## Camada Network (Mirror)

Aqui a regra é: **cada script de rede é fino, delega pro Core.**

### `MatchNetworkManager.cs`
- Herda de `Mirror.NetworkManager`
- Usa `NetworkManagerHUD` (componente pronto do Mirror) na `MainMenu.unity` pra tela de Host/Conectar por IP — evita reescrever essa UI do zero
- Spawna 2 `PlayerNetwork` (um por conexão), cada um vinculado a um `BoardEngine` distinto (esquerdo/direito na tela)

### `PlayerNetwork.cs` (NetworkBehaviour)
- Captura input local (`WASD`/setas) SÓ no cliente que tem autoridade (`isLocalPlayer`)
- Envia direção via `[Command]` pro host
- Host roda o `BoardEngine.Move()` correspondente e sincroniza o resultado via `[SyncVar]` ou `[ClientRpc]`

### `BoardNetworkSync.cs`
- Replica a lista de segmentos da cobra (posições) pros dois clientes
- Idealmente usando `SyncList<GridPosition>` do Mirror ou um `[ClientRpc]` disparado a cada tick de movimento

### `ItemNetworkSync.cs`
- Sincroniza posições de itens no tabuleiro (spawn/despawn)
- Bomba: `[Command] CmdThrowBomb(target)` → host valida e roda `BombSystem`, depois `[ClientRpc] RpcBombExploded(position)` pra ambos os clientes tocarem o efeito visual

**Ponto de atenção de autoridade:** o **host sempre roda a lógica de verdade** (server-authoritative) — os clientes só mandam intenção (direção, "jogar bomba", "escolher buff") e recebem o resultado já calculado. Isso evita que os dois lados fiquem "dessincronizados" e é o padrão recomendado do Mirror para jogos competitivos simples.

---

## Plano de implementação — 3 dias (multiplayer + IA solo)

> ⚠️ **Nota de escopo:** incluir a IA com A* real no mesmo prazo do multiplayer é ambicioso. O plano abaixo já reserva tempo pra isso, mas se o Dia 2 atrasar, a válvula de escape é: entregar a IA com heurística simples (sem A*, só "anda na direção geral da comida, desvia de colisão iminente") e documentar o A* completo como próxima melhoria — nunca cortar testes ou a sincronização de rede pra "salvar tempo" na IA.

### Dia 1 — Core + testes + rede básica
- [ ] `GridPosition`, `SnakeState`, `BoardEngine` (movimento, wraparound, colisão com corpo)
- [ ] Testes unitários do `BoardEngine` (mover, colidir, wraparound)
- [ ] Setup Mirror: `MatchNetworkManager`, cena `MainMenu` com `NetworkManagerHUD`
- [ ] `PlayerNetwork` básico: host e client conectam, cada um controla sua cobra, movimento sincroniza
- **Meta do dia:** duas cobras se movendo cada uma no seu tabuleiro, via rede, sem itens ainda.

### Dia 2 — Itens, bomba, buff/debuff + Core da IA
- [ ] `ItemSpawner` (comida comum + maçã bufada) sincronizado
- [ ] `SkillOrbPromptUI` — prompt de 2s pra escolher buff/debuff, com timeout
- [ ] `BombSystem` completo (fuse, área 3x3, penalidade) + sincronização via RPC
- [ ] Aplicar buff (velocidade+50%/5s) e debuff (inverte controle/5s) via RPC no alvo certo
- [ ] `AStarPathfinder.cs` (busca genérica, testável isoladamente) + testes unitários (caminho direto, desvio de obstáculo, wraparound)
- [ ] `SnakeAI.cs` básico: escolha de alvo (comida mais próxima) + integração com `BoardEngine`
- **Meta do dia:** loop de jogo multiplayer completo jogável; IA já anda até comida evitando o próprio corpo.

### Dia 3 — IA completa, timer, HUD, polimento
- [ ] `SnakeAI`: tratar bombas armadas como obstáculo temporário no pathfinding
- [ ] `SnakeAI`: heurísticas de orbe (buff/debuff) e bomba (joga perto do jogador)
- [ ] Menu "Modo Solo (vs IA)" além do multiplayer host-join
- [ ] Timer de 2 min + sudden death de 10s em caso de empate
- [ ] `HUDController`: placar, tempo restante, item na mão
- [ ] `EndScreenUI`: tela de vitória/derrota
- [ ] Sprites simples (cobra, itens) no lugar dos placeholders
- [ ] README documentando arquitetura Core/Network + decisões (host-join manual, sem infra paga, IA com A*)
- **Meta do dia:** build jogável ponta a ponta (multiplayer + solo vs IA), pronto pra portfólio e publicação no itch.io.

---

## Notas de design a lembrar durante a implementação

- Velocidade: 180ms/tick no início, -15ms a cada 30s (180→165→150→135)
- Dano de bomba só afeta a **cabeça** — corpo é imune
- Pontuação sempre em múltiplos de 10 (evita camada de "display" separada)
- Cobra começa com 3 segmentos, centro do próprio tabuleiro
- Tabuleiro 24x24 (mesmo valor será reusado no Modo 2 no futuro)
