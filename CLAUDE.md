# App "Aos Pés do Mestre Jesus" (discipulado)

## Identidade deste projeto
- Pasta local neste PC: `aos-pes-do-mestre-jesus`
- Repositório GitHub: `capelaniahospitalar/jornada-discipular` (o nome do repo NÃO bate com o da pasta — é histórico; confirmar sempre por `git remote -v`)
- Produto: app de discipulado — estudos bíblicos, gamificação (nível/XP), diário espiritual, painel do tutor
- Publicado em: capelaniahospitalar.github.io/jornada-discipular/

## Separação de produtos (decisão de 2026-07-08)
- Este app é INDEPENDENTE do app de Pequenos Grupos (repo `capelaniahospitalar/jornada-pequenos-grupos`), apesar do histórico Git comum.
- NUNCA copiar funcionalidades, nomenclaturas ou decisões do app de Pequenos Grupos para cá sem pedido explícito do usuário.
- Se uma funcionalidade parecer pertencer ao outro produto, PARAR e perguntar antes de implementar.

## Antes de qualquer trabalho (obrigatório)
1. Rodar `git fetch origin` e `git status -sb`.
2. Se houver atualizações do GitHub para baixar (behind), fazer `git pull` ANTES de editar qualquer arquivo — o usuário trabalha em mais de um computador e a versão mais recente pode ter sido enviada pelo outro PC.
3. Se houver alterações locais não commitadas de sessão anterior, avisar o usuário antes de prosseguir.

## Sobre o usuário e publicação
- O usuário não é programador: explicar decisões em linguagem simples e confirmar escolhas de design antes de aplicar.
- Publicação via GitHub Desktop (conta capelaniahospitalar) ou upload manual pela web; `git push` direto do terminal pode falhar com erro 403.
