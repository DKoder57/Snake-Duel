# Snake Duel — Setup e Prevenção de Problemas

> Companheiro do `snake-duel-arquitetura-tecnica.md`. Este documento cobre a ORDEM de configuração e os erros mais comuns ao montar um projeto Unity + Mirror do zero — ler antes de criar o projeto, não depois de já ter tropeçado neles.

## Ordem correta de setup (importa)

Fazer na ordem errada é a causa mais comum de retrabalho. Siga assim:

1. **`git init` ANTES de abrir o Unity pela primeira vez.**
   Se você criar o projeto Unity primeiro e só depois lembrar do Git, o primeiro `git add .` vai capturar a pasta `Library/` inteira (pode ser 500MB+) antes do `.gitignore` existir. Limpar isso depois exige reescrever histórico. Ordem certa: `git init` → criar `.gitignore` → só então abrir/criar o projeto no Unity Hub.

2. **Configurar serialização ANTES do primeiro asset ser criado.**
   `Edit → Project Settings → Editor`:
   - `Version Control Mode` → `Visible Meta Files`
   - `Asset Serialization Mode` → `Force Text`

   Se você mudar isso depois de já ter várias cenas/prefabs criados em modo binário, o Unity reserializa tudo — geralmente funciona, mas é um passo a mais e um commit gigante de "reformatação" que polui o histórico. Mais barato acertar antes do primeiro asset.

3. **Instalar o Mirror via Package Manager (Git URL), não copiando arquivos manualmente.**
   `Window → Package Manager → + → Add package from git URL` com a URL oficial do repositório do Mirror. Isso garante que você recebe atualizações via Package Manager em vez de ter uma cópia "congelada" difícil de atualizar depois.

## Pitfalls específicos do Mirror

### "Two NetworkManagers" ao testar localmente
Ao testar host + client na mesma máquina (rodando Editor + um Build em paralelo, ou duas instâncias do Editor), é comum acidentalmente deixar dois objetos `NetworkManager` ativos ao mesmo tempo na mesma cena carregada duas vezes — isso gera erros confusos de "already spawned" ou conexões fantasmas. Mirror já lida com isso via singleton internamente, mas o erro típico vem de **cena de rede duplicada em builds/editor rodando ao mesmo tempo sem isolamento correto de porta**.
**Prevenção:** para testar 2 jogadores na mesma máquina, use um Build separado (não duas janelas do Editor) rodando ao lado do Editor em Play Mode — um vira host, o outro conecta em `127.0.0.1`.

### Player Prefab não registrado
Se o `PlayerNetwork` prefab não estiver arrastado no campo `Player Prefab` do `NetworkManager` (ou registrado via `NetworkManager.playerPrefab`), a conexão acontece mas nenhuma cobra aparece — sem erro óbvio no console, só "nada acontece". É o bug mais comum de quem está começando com Mirror.
**Prevenção:** checklist de setup do `NetworkManager`: Player Prefab atribuído + prefab tem `NetworkIdentity` component.

### Cena Offline vs Online
Mirror espera uma cena "offline" (menu, antes de conectar) e uma "online" (partida em andamento) configuradas no `NetworkManager`. Esquecer de configurar isso corretamente faz o jogo trocar de cena de forma inesperada ao conectar/desconectar.
**Prevenção:** configurar `Offline Scene` = `MainMenu`, `Online Scene` = `Match`, logo na primeira sessão de setup.

## Pitfalls gerais de Unity + Git

### Merge de cena/prefab
Já mencionado na conversa anterior, mas reforçando aqui como checklist: mesmo com `Force Text`, evite ter a mesma cena (`Match.unity`) modificada em duas branches abertas simultaneamente. Termine e commite uma feature antes de abrir a próxima branch que mexe na mesma cena.

### Prefab overrides "fantasmas"
Editar uma instância de prefab direto na cena (em vez de entrar no modo de edição do prefab) cria "overrides" que ficam salvos na cena, não no prefab — depois é fácil esquecer que aquela instância específica está diferente das outras. Prefira sempre editar prefabs no Prefab Mode (duplo clique no asset), não na cena.

### Assets binários grandes sem Git LFS
Sprites simples (poucos KB) não são problema, mas se você adicionar qualquer áudio, sprite sheet maior ou build de teste commitado por engano, o repositório cresce permanentemente (Git não "esquece" arquivos binários grandes do histórico sem reescrever). Ativar Git LFS cedo, mesmo que não pareça necessário ainda, é mais barato do que migrar depois.

## Testando o Core isoladamente (sem depender do Mirror)

Com os `.asmdef` da arquitetura já configurados, rode os testes via `Window → General → Test Runner → EditMode`. Isso executa `BoardEngineTests`, `BombSystemTests`, `AStarPathfinderTests` sem precisar entrar em Play Mode nem ter o Mirror ativo — é o retorno prático de ter investido na separação Core/Network desde o início.

## Build para itch.io

- **Target platforms:** gerar builds separados de Windows, Mac e Linux (`File → Build Settings`) — itch.io aceita os três num único "app" com uploads distintos por plataforma.
- **Backend de scripting:** para builds de desktop simples como este, Mono é suficiente; IL2CPP só passa a importar se for pra outras plataformas (mobile/console) no futuro.
- Sem build WebGL — motivo já documentado no README principal (modelo host-and-join não funciona em navegador).
