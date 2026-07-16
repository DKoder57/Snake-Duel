# Snake Duel

Snake competitivo 1v1 (host-and-join direto por IP) com modo solo contra IA, inspirado em Slither.io e Tetris Battle. Protótipo desenvolvido como projeto de portfólio, com foco em arquitetura testável e multiplayer sem infraestrutura paga.


## Features

- **Modo 1 (Tempo Limite)** — dois tabuleiros lado a lado, 2 minutos, vence quem tiver mais pontos. Wraparound nas paredes, itens (comida, maçã bufada, orbe de habilidade, bomba), buff/debuff temporário.
- **Modo Solo (vs IA)** — mesma regra do Modo 1, adversário controlado por uma IA com pathfinding A* real.
- **Modo 2 (Sobrevivência)** — documentado, não implementado nesta versão (ver seção "Roadmap").

## Arquitetura

O projeto separa lógica de jogo (`Core`) de sincronização de rede (`Network`), permitindo testar toda a lógica (movimento, colisão, itens, IA) sem depender do Mirror rodando. Ver `snake-duel-arquitetura-tecnica.md` para detalhes completos de cada script.

**Rede:** Mirror Networking, modelo host-and-join direto por IP (sem servidor dedicado, sem custo de infraestrutura) — o mesmo modelo usado por jogos como Counter-Strike clássico.

**IA:** A* real sobre grid toroidal (o wraparound é resolvido nativamente pela aritmética modular de `GridPosition`, sem código extra de busca). Bombas armadas são tratadas como obstáculos temporários no grafo de busca.

## Como jogar

1. Um jogador clica "Hospedar" na tela inicial.
2. O outro jogador digita o IP do host e clica "Conectar".
3. Para partidas fora da rede local, o host precisa configurar port forwarding no roteador (ver seção "Problemas conhecidos" abaixo).

## Publicação

Build de download (Windows/Mac/Linux) via itch.io. **Não há versão WebGL** — o modelo host-and-join exige abrir uma porta de rede, algo que navegadores não permitem por sandboxing/segurança, e mixed content bloquearia conexão não-criptografada a partir de uma página HTTPS.

## 💻 Tech Stack

| Camada | Tecnologia / Ferramenta | Descrição / Uso |
| :--- | :--- | :--- |
| **Engine Principal** | **Unity (C#)** | Desenvolvimento do ciclo de vida do jogo, renderização e gerenciamento de estados. |
| **Rede & Multiplayer** | **Mirror Networking** | Framework de alto nível para gerenciamento de pacotes, sincronização de estados e arquitetura Host/Client baseada em IP. |
| **Algoritmos & IA** | **$A^*$ Pathfinding (Customizado)** | Algoritmo genérico de busca de menor caminho para navegação e tomadas de decisão da Inteligência Artificial. |
| **Arquitetura de Software**| **POCO (Plain Old C# Objects)** | Lógica de jogo limpa e desacoplada da Unity (`MonoBehaviour`) para o *BoardEngine*. |
| **Qualidade & Testes** | **Unity Test Framework (NUnit)** | Execução de testes unitários automatizados para validação de regras de física lógica, pontuação e detecção de colisões. |
| **Automação & CI/CD** | **GitHub Actions** | Integração contínua para gerenciamento de fluxo de trabalho e automações do repositório. |

---

## ⚠️ Problemas conhecidos e riscos futuros

Esta seção existe para deixar claro, tanto pra mim quanto pra quem for avaliar o projeto, que essas limitações são decisões conscientes de escopo — não bugs não percebidos.

### 1. CGNAT pode inviabilizar o host completamente (risco mais sério)
Muitos provedores de internet no Brasil (e no mundo) usam **CGNAT (Carrier-Grade NAT)** — o roteador do usuário não tem IP público de verdade, só um IP interno atrás do NAT do próprio provedor. Nesse cenário, **port forwarding simplesmente não funciona**, porque não existe uma porta pública real pra abrir. Isso significa que uma parcela dos usuários (especialmente em conexões residenciais/móveis mais novas) não vai conseguir hospedar partida nenhuma pela internet, só em rede local.
**Mitigação futura:** migrar pra um transporte com NAT traversal assistido (ex: relay via Steam P2P quando for pra Steam, ou um serviço de relay leve tipo Mirror + Edgegap/Photon) — mas isso reintroduz a dependência de infraestrutura que decidimos evitar nesta fase.

### 2. Velocidade dinâmica pode causar dessincronia sutil
O tick de movimento acelera ao longo da partida (180ms → 135ms). Se o host e o cliente não recalcularem o intervalo exatamente no mesmo instante (drift de poucos milissegundos por lag de rede), a sensação de "responsividade" pode divergir entre os dois lados perto das transições de velocidade.
**Mitigação futura:** logar timestamp do host como fonte única de verdade e forçar re-sync do timer de velocidade a cada RPC, não só no clock local do cliente.

### 3. Bomba e explosão perto da borda do tabuleiro (wraparound)
A área de explosão 3x3 precisa considerar que uma bomba perto da borda "vaza" pro lado oposto do grid (efeito do wraparound). Se o cálculo de área não usar a mesma aritmética modular do `GridPosition`, bombas nas bordas podem ter área de efeito incorreta (menor do que deveria, ou não detectar a cabeça da cobra corretamente).
**Mitigação:** cobrir esse caso especificamente nos testes unitários do `BombSystem` (bomba em `x=0`, `x=23`, cantos).

### 4. IA sem caminho válido (grid muito cheio)
Perto do fim da partida, com cobras grandes e vários itens/bombas ocupando espaço, pode não existir caminho livre até nenhum alvo. O A* precisa de um **fallback definido** (ex: continuar na direção atual, ou buscar a célula livre mais próxima) — sem isso, a IA pode travar ou lançar exceção.
**Mitigação:** já prever esse fallback na implementação do Dia 2/3, não deixar para depois.

### 5. Escolha de buff/debuff com timeout de rede
O prompt de 2 segundos pra escolher buff/debuff depende de o input do jogador chegar ao host antes do timeout. Em conexões com latência alta, o jogador pode ver a janela de escolha mas o host já ter processado o timeout quando o clique chegar — gerando a sensação de "não registrou minha escolha".
**Mitigação futura:** considerar folga extra no timeout do lado do host (ex: 2.3s no servidor vs 2s exibido no cliente).


## Roadmap

- [ ] Modo 2 (Sobrevivência): espinhos estáticos, sistema de vidas, maçã dourada
- [ ] IA com previsão de movimento do jogador ao jogar bombas (hoje mira na posição atual, não na futura)
- [ ] Avaliação de transporte com NAT traversal assistido, caso CGNAT se mostre um bloqueio frequente em testes reais
