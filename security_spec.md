# Especificação de Segurança e Auditoria Firestore - PMPI Checklists

## 1. Invariantes de Dados (Data Invariants)
1. **Identidade e Autorização**: Um usuário NUNCA pode conceder a si mesmo o perfil de Administrador Geral ou aprovar a si mesmo. Novos registros são forçados para `status: "pendente"` e `role: "operacional" | "administrativo"` (ou perfil pretendido neutro sem privilégios).
2. **Isolamento de Batalhão (Multi-Tenancy por Unidade)**:
   - Administradores de Batalhão (`role == "administrativo"`) têm permissão de leitura estritamente restrita à sua própria unidade (`unidadeId == user.unidadeId`). Não podem ler nem alterar dados de outros batalhões.
   - Policiais Operacionais (`role == "operacional"`) podem criar e enviar checklists apenas para a unidade à qual pertencem e ler seus próprios envios.
   - Administrador Geral (`isAdmin()`) tem visão global de todas as unidades e autoridade exclusiva para aprovar/rejeitar e gerenciar usuários.
3. **Bloqueio de Usuários Pendentes**: Usuários com `status != "aprovado"` são proibidos de realizar qualquer leitura ou gravação de checklists operacionais.
4. **Imutabilidade de Histórico**: Um checklist aprovado/finalizado não pode ter seu autor (`responsavelUid`), sua unidade (`unidadeId`) ou sua data original (`createdAt`) adulterados.
5. **Proteção Contra Injeção e IDs Inválidos**: Todos os IDs devem respeitar regex `^[a-zA-Z0-9_\-]+$` e limites máximos de tamanho.

## 2. As 12 Cargas Maliciosas de Teste (The Dirty Dozen Payloads)
1. **Self-Promotion Admin Exploit**: Usuário cria perfil inicial com `role: "admin_geral"` ou `status: "aprovado"`.
   - *Resultado esperado:* PERMISSION_DENIED.
2. **Cross-Battalion Snooping**: Administrador do 1º BPM tenta executar query ou leitura direta de checklists do 2º BPM.
   - *Resultado esperado:* PERMISSION_DENIED.
3. **Ghost Field Injection**: Usuário envia checklist com campo malicioso `__backdoor_role: "god"`.
   - *Resultado esperado:* PERMISSION_DENIED.
4. **Pending User Checklist Read**: Usuário cadastrado com status "pendente" tenta listar ou consultar checklists.
   - *Resultado esperado:* PERMISSION_DENIED.
5. **Pending User Checklist Write**: Usuário não aprovado tenta enviar checklist operacional.
   - *Resultado esperado:* PERMISSION_DENIED.
6. **Responsavel Forgery**: Usuário autenticado tenta enviar checklist com `responsavelUid` apontando para outro policial militar.
   - *Resultado esperado:* PERMISSION_DENIED.
7. **Unidade Mismatch Attack**: Policial do 4º BPM tenta registrar viatura como pertencente ao BOPE.
   - *Resultado esperado:* PERMISSION_DENIED.
8. **Tamper With CreatedAt**: Usuário tenta falsificar data de auditoria (`createdAt: "2010-01-01"`).
   - *Resultado esperado:* PERMISSION_DENIED.
9. **Role Escalation in Update**: Operacional tenta atualizar seu próprio documento `/users/{uid}` para alterar `role` para `admin_geral`.
   - *Resultado esperado:* PERMISSION_DENIED.
10. **ID Poisoning Attack**: Criação de documento com ID malicioso de 2000 caracteres ou injeção SQL/caracteres proibidos.
    - *Resultado esperado:* PERMISSION_DENIED.
11. **Checklist Delete by Non-Admin**: Policial operacional tenta apagar checklist já submetido.
    - *Resultado esperado:* PERMISSION_DENIED.
12. **Unauthorized Unit Modification**: Usuário comum tenta criar ou desativar uma unidade da PMPI.
    - *Resultado esperado:* PERMISSION_DENIED.

## 3. Segurança do Recurso "Viaturas" (Subcoleção units/{unidadeId}/viaturas/{viaturaId})
1. **Invariantes do Recurso de Viaturas**:
   - Administrador do Batalhão (`role == "administrativo"`) tem permissão de escrita (create, update, delete) exclusivamente nas viaturas da sua própria unidade.
   - Administrador Geral (`isAdmin()`) pode ler e escrever em todas as unidades.
   - Policial Operacional (`role == "operacional"`) pode apenas ler as viaturas da sua própria unidade; escrita é estritamente proibida.
   - Viatura inoperante (`status == "inoperante"`) exige obrigatoriamente um motivo (`motivoInoperancia`) com tamanho mínimo de 5 caracteres.
   - Tentativa de criar viatura sem motivo quando inoperante ou por perfil operacional resulta em `PERMISSION_DENIED`.

## 4. Isolamento Estrito de Vistorias por Policial (Visibilidade Operacional)
1. **Invariantes de Acesso a Vistorias (`units/{unidadeId}/checklists/{checklistId}`)**:
   - **Operacional (`role == "operacional"`)**:
     * Leitura individual (`get`): Permitido exclusivamente se `policialId` (ou `responsavelUid`) corresponder a `request.auth.uid`. Tentar ler vistoria de colega da mesma unidade resulta em `PERMISSION_DENIED`.
     * Listagem (`list`): Permitido exclusivamente com consulta filtrada (`where('policialId', '==', request.auth.uid)`). Tentativa de listar a unidade sem o filtro de policial resulta em `PERMISSION_DENIED`.
   - **Administrador do Batalhão (`role == "administrativo"`)**:
     * Leitura individual e listagem: Permitido para todas as vistorias pertencentes ao seu próprio batalhão (`unidadeId == user.unidadeId`).
     * Tentativa de ler ou listar vistorias de outro batalhão resulta em `PERMISSION_DENIED`.
   - **Administrador Geral (`role == "admin_geral"`)**:
     * Leitura e listagem de qualquer vistoria de qualquer unidade ou via collection group global (`PERMISSION_GRANTED`).
