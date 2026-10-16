# VittaCare — Dashboard Executivo (PBIP)

Projeto Power BI gerado em formato **PBIP** (modelo em TMDL + relatório em PBIR). Abra `VittaCare.pbip` no Power BI Desktop.

## 1. Arquitetura (Star Schema)

```
                 Dim_Calendario (Data)            Dim_Clinicas (ID_Clinica)
                          \  1:*                    1:*  /
 Dim_Pacientes (ID_Paciente) ——1:*——  Fato_Atendimentos  ——*:1—— Dim_Medicos (ID_Medico)
                                             |
                                            *:1
                                   Dim_Especialidades (ID_Especialidade)
```
- `ID_Especialidade` é trazido para a fato no Power Query (merge com Dim_Medicos) → modelo em estrela puro, sem snowflake.
- `Dim_Calendario` é gerada em Power Query (2024–2025), marcada como tabela de datas, meses em pt-BR ordenados.
- `_Medidas` é a tabela só de medidas (pastas 01 a 06).
- Parâmetro `PastaDados` = `E:\Projetos BI\11 - Vita Care\` (Transformar dados › Gerir parâmetros, se mover a pasta).
- A base **não tem idade/género**, por isso a Aba 3 mostra **Status por Plano de Saúde** em vez de perfil demográfico.

## 2. Paleta / Tema (`VittaCare_Theme.json`, já aplicado)

| Token | Hex | Uso |
|---|---|---|
| Verde Esmeralda | `#00CBA9` | série principal, positivo |
| Esmeralda escuro | `#00A88C` | textos positivos |
| Azul Corporativo | `#2C3E50` | comparação, títulos |
| Cinza Texto | `#64748B` | rótulos, tabelas |
| Cinza Nuvem | `#F4F7F9` | fundo da página |
| Cabeçalho tabela | `#F1F5F9` | cabeçalhos centralizados |
| Linha alternada | `#F8FAFC` | zebra |
| Coral (atenção) | `#E8505B` | cancelamento acima da meta |
| Âmbar | `#F5A623` | neutro/no-show |

## 3. Layout das 3 abas (canvas 1280×720, fundo `VittaCare_Background.png`)

**Cabeçalho comum (y 0–72):** logo (x24) · 3 botões de navegação (x268–736, aba ativa em verde suave) · segmentações Ano / Mês / Unidade (x800–1260, sincronizadas entre páginas).

| Aba | Linha 1 | Linha 2 | Linha 3 |
|---|---|---|---|
| **1 · Visão Executiva** | Banner HTML "Portal Executivo" (y84, largura total) | 4 cartões HTML: Faturamento · Consultas · Cancelamento · Ticket Médio (y170) | Deneb Área "Evolução do Faturamento" (x24, 744×386) + coluna direita: Deneb Barras "Faturamento por Unidade" e Deneb Donut "Consultas por Especialidade" (472×186 cada) |
| **2 · Desempenho Clínico** | 4 cartões HTML: Nota Média · Tempo de Espera · Consultas Realizadas · Pacientes Únicos (y80) | Deneb Scatter "Tempo de Espera × Nota" (x24, 604×474) | Deneb Lollipop "Ranking de Médicos" + Deneb Colunas "Receita por Especialidade" (612×~230 cada) |
| **3 · Detalhe Operacional** | 4 cartões HTML: Agendamentos · Receita Convênios · Receita Particular · No-Show (y80) | Deneb 100% "Status por Plano" (496×244) + Tabela "Resumo por Plano" (720×244) | Tabela "Detalhamento de Atendimentos" (largura total, exportável) |

## 4. Medidas DAX essenciais

Todas já estão em `_Medidas`. Principais:

```dax
Faturamento Total =
SUM ( Fato_Atendimentos[Valor Cobrado] )

Faturamento Ano Atual =
VAR _ano = MAX ( Dim_Calendario[Ano] )
RETURN
    CALCULATE ( [Faturamento Total], Dim_Calendario[Ano] = _ano )

Faturamento Ano Anterior =
VAR _ano = MAX ( Dim_Calendario[Ano] )
RETURN
    CALCULATE ( [Faturamento Total], Dim_Calendario[Ano] = _ano - 1 )

Faturamento Var % AA =
VAR _atual = [Faturamento Ano Atual]
VAR _anterior = [Faturamento Ano Anterior]
RETURN
    IF ( NOT ISBLANK ( _anterior ), DIVIDE ( _atual - _anterior, _anterior ) )

Agendamentos =
COUNTROWS ( Fato_Atendimentos )

Consultas Realizadas =
CALCULATE ( COUNTROWS ( Fato_Atendimentos ), KEEPFILTERS ( Fato_Atendimentos[Status Consulta] = "Realizada" ) )

Consultas Canceladas =
CALCULATE ( COUNTROWS ( Fato_Atendimentos ), KEEPFILTERS ( Fato_Atendimentos[Status Consulta] = "Cancelada" ) )

Consultas No-Show =
CALCULATE ( COUNTROWS ( Fato_Atendimentos ), KEEPFILTERS ( Fato_Atendimentos[Status Consulta] = "No-Show" ) )

Consultas Realizadas Var % AA =
VAR _ano = MAX ( Dim_Calendario[Ano] )
VAR _atual = CALCULATE ( [Consultas Realizadas], Dim_Calendario[Ano] = _ano )
VAR _anterior = CALCULATE ( [Consultas Realizadas], Dim_Calendario[Ano] = _ano - 1 )
RETURN
    IF ( NOT ISBLANK ( _anterior ), DIVIDE ( _atual - _anterior, _anterior ) )

% Cancelamento =
DIVIDE ( [Consultas Canceladas], [Agendamentos] )

% No-Show =
DIVIDE ( [Consultas No-Show], [Agendamentos] )

Meta Cancelamento =
0.10

Ticket Médio =
DIVIDE ( [Faturamento Total], [Consultas Realizadas] )

Ticket Médio Var % AA =
VAR _ano = MAX ( Dim_Calendario[Ano] )
VAR _atual = CALCULATE ( [Ticket Médio], Dim_Calendario[Ano] = _ano )
VAR _anterior = CALCULATE ( [Ticket Médio], Dim_Calendario[Ano] = _ano - 1 )
RETURN
    IF ( NOT ISBLANK ( _anterior ), DIVIDE ( _atual - _anterior, _anterior ) )

Tempo Espera Médio =
CALCULATE ( AVERAGE ( Fato_Atendimentos[Tempo Espera (min)] ), KEEPFILTERS ( Fato_Atendimentos[Status Consulta] = "Realizada" ) )

Meta Tempo Espera =
30

% Dentro SLA Espera =
DIVIDE (
    CALCULATE (
        COUNTROWS ( Fato_Atendimentos ),
        KEEPFILTERS ( Fato_Atendimentos[Status Consulta] = "Realizada" ),
        KEEPFILTERS ( Fato_Atendimentos[Tempo Espera (min)] <= 30 )
    ),
    [Consultas Realizadas]
)

Nota Média =
AVERAGE ( Fato_Atendimentos[Nota Atendimento] )

Qtd Avaliações =
COUNT ( Fato_Atendimentos[Nota Atendimento] )

Pacientes Únicos =
DISTINCTCOUNT ( Fato_Atendimentos[ID_Paciente] )

Cor Status =
SWITCH (
    SELECTEDVALUE ( Fato_Atendimentos[Status Consulta] ),
    "Realizada", "#00A88C",
    "Cancelada", "#E8505B",
    "No-Show", "#E59A1F",
    "#64748B"
)

Cor Fundo Cancelamento =
IF ( [% Cancelamento] > [Meta Cancelamento], "#FDECEE", "#E6FAF6" )

Cor Fonte Cancelamento =
IF ( [% Cancelamento] > [Meta Cancelamento], "#C0392B", "#00866F" )

Cor Espera =
IF ( SUM ( Fato_Atendimentos[Tempo Espera (min)] ) > [Meta Tempo Espera], "#E8505B", "#64748B" )
```

## 5. Cartões HTML animados (visual **HTML Content**)

Cada cartão é uma medida DAX que devolve HTML + CSS: hover levanta o cartão (`translateY(-4px)` + sombra), ícone SVG pulsa (`@keyframes pulse`) ou roda (`spin`), o valor entra com fade-up, as barras de meta crescem (`grow`) e a cor muda para coral quando a meta é ultrapassada.

### HTML Card Faturamento
```dax
HTML Card Faturamento =
VAR _v = [Faturamento Var % AA]
VAR _ano = MAX ( Dim_Calendario[Ano] )
VAR _pill =
    IF (
        ISBLANK ( _v ),
        "<span class='pill' style='background:#F1F5F9;color:#64748B'>sem base</span><span>ano anterior</span>",
        "<span class='pill' style='background:" & IF ( _v >= 0, "rgba(0,203,169,0.14);color:#00A88C'>▲ +", "rgba(232,80,91,0.12);color:#D64550'>▼ -" )
            & FORMAT ( ABS ( _v ), "0.0%", "pt-BR" ) & "</span><span>" & _ano & " vs. " & ( _ano - 1 ) & "</span>"
    )
VAR _acc = "#00CBA9"
VAR _rgb = "0,203,169"
VAR _valor = IF ( ISBLANK ( [Faturamento Total] ), "—", "R$ " & IF ( [Faturamento Total] >= 1000000, FORMAT ( [Faturamento Total] / 1000000, "#,0.0", "pt-BR" ) & " mi", IF ( [Faturamento Total] >= 1000, FORMAT ( [Faturamento Total] / 1000, "#,0.0", "pt-BR" ) & " mil", FORMAT ( [Faturamento Total], "#,0", "pt-BR" ) ) ) )
VAR _css = "<style>*{box-sizing:border-box;margin:0;padding:0}html,body{background:transparent;overflow:hidden}.card{font-family:Inter,'Segoe UI',Arial,sans-serif;height:calc(100vh - 18px);margin:6px 8px 12px 8px;background:rgba(255,255,255,0.88);backdrop-filter:blur(10px);border:1px solid rgba(226,232,240,0.95);border-radius:16px;padding:14px 18px 12px 22px;box-shadow:0 2px 10px rgba(44,62,80,0.06);transition:transform .35s cubic-bezier(.2,.8,.2,1),box-shadow .35s ease,border-color .35s ease;position:relative;overflow:hidden;display:flex;flex-direction:column;justify-content:space-between;cursor:default}.card:hover{transform:translateY(-4px);box-shadow:0 14px 28px rgba(44,62,80,0.14);border-color:rgba(" & _rgb & ",0.55)}.card:before{content:'';position:absolute;left:0;top:16px;bottom:16px;width:4px;border-radius:0 4px 4px 0;background:" & _acc & "}.card:after{content:'';position:absolute;right:-46px;top:-46px;width:130px;height:130px;border-radius:50%;background:radial-gradient(circle,rgba(" & _rgb & ",0.13),rgba(" & _rgb & ",0) 70%);transition:transform .6s ease}.card:hover:after{transform:scale(1.5)}.top{display:flex;justify-content:space-between;align-items:center;position:relative;z-index:1}.lbl{font-size:11px;font-weight:600;color:#64748B;letter-spacing:.6px;text-transform:uppercase}.ico{width:34px;height:34px;border-radius:10px;background:rgba(" & _rgb & ",0.12);display:flex;align-items:center;justify-content:center;flex-shrink:0}.ico svg{width:19px;height:19px;stroke:" & _acc & ";fill:none;stroke-width:2;stroke-linecap:round;stroke-linejoin:round}.pulse{animation:pulse 2.6s ease-in-out infinite}.spin svg{animation:spin 7s linear infinite}.beat svg polyline{stroke-dasharray:64;stroke-dashoffset:64;animation:draw 2.4s ease-in-out infinite}@keyframes pulse{0%,100%{box-shadow:0 0 0 0 rgba(" & _rgb & ",0.35);transform:scale(1)}50%{box-shadow:0 0 0 7px rgba(" & _rgb & ",0);transform:scale(1.07)}}@keyframes spin{to{transform:rotate(360deg)}}@keyframes draw{0%{stroke-dashoffset:64}50%{stroke-dashoffset:0}100%{stroke-dashoffset:-64}}@keyframes up{from{opacity:0;transform:translateY(10px)}to{opacity:1;transform:none}}@keyframes grow{from{width:0}}.val{font-size:30px;font-weight:700;letter-spacing:-.5px;line-height:1.1;position:relative;z-index:1;animation:up .9s cubic-bezier(.2,.8,.2,1) both}.val small{font-size:11px;font-weight:500;color:#64748B;letter-spacing:0;margin-left:6px}.foot{font-size:11px;color:#64748B;display:flex;gap:8px;align-items:center;white-space:nowrap;overflow:hidden;position:relative;z-index:1}.pill{font-weight:700;padding:2px 8px;border-radius:20px;font-size:10.5px}.bar{flex:1;min-width:60px;height:6px;border-radius:6px;background:#EEF2F6;position:relative;display:flex;overflow:visible}.bar i{display:block;height:100%;border-radius:6px;animation:grow 1.4s cubic-bezier(.2,.8,.2,1) both}.bar b{position:absolute;top:-3px;width:2px;height:12px;border-radius:2px;background:#2C3E50}.stars{position:relative;display:inline-block;color:#E2E8F0;font-size:13px;letter-spacing:2px}.stars span{position:absolute;left:0;top:0;overflow:hidden;white-space:nowrap;color:#F5B700;animation:grow 1.6s ease-out both}.more{position:absolute;inset:0;z-index:5;border-radius:16px;padding:11px 16px 9px 22px;color:#fff;background:linear-gradient(150deg,rgba(44,62,80,0.97) 0%,rgba(31,45,58,0.97) 60%,rgba(27,59,61,0.97) 100%);opacity:0;transform:translateY(14px) scale(.98);transition:opacity .35s ease,transform .45s cubic-bezier(.2,.8,.2,1);pointer-events:none;display:flex;flex-direction:column;justify-content:space-between}.more:before{content:'';position:absolute;left:0;top:16px;bottom:16px;width:4px;border-radius:0 4px 4px 0;background:" & _acc & "}.card:hover .more{opacity:1;transform:none}.mt{font-size:9.5px;font-weight:700;letter-spacing:.9px;text-transform:uppercase;color:#33D6BB;display:flex;justify-content:space-between}.mt em{font-style:normal;color:#94A3B8;font-weight:500;letter-spacing:.3px;text-transform:none}.mg{display:grid;grid-template-columns:1fr 1fr;gap:3px 14px}.mg div{display:flex;justify-content:space-between;align-items:baseline;gap:6px;border-bottom:1px dashed rgba(255,255,255,0.10);padding-bottom:2px;opacity:0;transform:translateY(6px);transition:opacity .4s ease,transform .4s ease}.card:hover .mg div{opacity:1;transform:none}.card:hover .mg div:nth-child(2){transition-delay:.06s}.card:hover .mg div:nth-child(3){transition-delay:.12s}.card:hover .mg div:nth-child(4){transition-delay:.18s}.mg span{font-size:9.5px;color:#94A3B8;white-space:nowrap}.mg b{font-size:12px;font-weight:700;color:#fff;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.mi{font-size:10px;color:#E2E8F0;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;opacity:0;transition:opacity .5s .25s ease}.mi b{color:#33D6BB}.card:hover .mi{opacity:1}.hint{position:absolute;right:10px;bottom:6px;font-size:8.5px;color:#CBD5E1;letter-spacing:.3px;z-index:2;transition:opacity .3s}.card:hover .hint{opacity:0}</style>"
RETURN
    _css
        & "<div class='card'><div class='top'><span class='lbl'>Faturamento Total</span>"
        & "<div class='ico pulse'><svg viewBox='0 0 24 24'><line x1='12' y1='2' x2='12' y2='22'/><path d='M17 5H9.5a3.5 3.5 0 0 0 0 7h5a3.5 3.5 0 0 1 0 7H6'/></svg></div></div>"
        & "<div class='val' style='color:" & "#00A88C" & "'>" & _valor & "</div>"
        & "<div class='foot'>" & _pill & "</div>"
        & "<div class='hint'>ⓘ detalhes</div>"
        & "<div class='more'><div class='mt'>Detalhe do faturamento</div><div class='mg'>" & "<div><span>Ano atual</span><b>" & ( IF ( ISBLANK ( [Faturamento Ano Atual] ), "—", "R$ " & IF ( [Faturamento Ano Atual] >= 1000000, FORMAT ( [Faturamento Ano Atual] / 1000000, "#,0.0", "pt-BR" ) & " mi", IF ( [Faturamento Ano Atual] >= 1000, FORMAT ( [Faturamento Ano Atual] / 1000, "#,0.0", "pt-BR" ) & " mil", FORMAT ( [Faturamento Ano Atual], "#,0", "pt-BR" ) ) ) ) ) & "</b></div>" & "<div><span>Ano anterior</span><b>" & ( IF ( ISBLANK ( [Faturamento Ano Anterior] ), "—", "R$ " & IF ( [Faturamento Ano Anterior] >= 1000000, FORMAT ( [Faturamento Ano Anterior] / 1000000, "#,0.0", "pt-BR" ) & " mi", IF ( [Faturamento Ano Anterior] >= 1000, FORMAT ( [Faturamento Ano Anterior] / 1000, "#,0.0", "pt-BR" ) & " mil", FORMAT ( [Faturamento Ano Anterior], "#,0", "pt-BR" ) ) ) ) ) & "</b></div>" & "<div><span>Média/mês</span><b>" & ( IF ( ISBLANK ( [Faturamento Média Mensal] ), "—", "R$ " & IF ( [Faturamento Média Mensal] >= 1000000, FORMAT ( [Faturamento Média Mensal] / 1000000, "#,0.0", "pt-BR" ) & " mi", IF ( [Faturamento Média Mensal] >= 1000, FORMAT ( [Faturamento Média Mensal] / 1000, "#,0.0", "pt-BR" ) & " mil", FORMAT ( [Faturamento Média Mensal], "#,0", "pt-BR" ) ) ) ) ) & "</b></div>" & "<div><span>Convênios</span><b>" & ( IF ( ISBLANK ( [% Receita Convênios] ), "—", FORMAT ( [% Receita Convênios], "0.0%", "pt-BR" ) ) ) & "</b></div>" & "</div>"
        & "<div class='mi'>" & ( "Unidade líder: <b>" & [Top Unidade Faturamento] & "</b>" & " · melhor mês <b>" & [Melhor Mês Faturamento] & "</b>" ) & "</div></div>"
        & "</div>"
```

### HTML Card Consultas
```dax
HTML Card Consultas =
VAR _v = [Consultas Realizadas Var % AA]
VAR _ano = MAX ( Dim_Calendario[Ano] )
VAR _pill =
    IF (
        ISBLANK ( _v ),
        "<span class='pill' style='background:#F1F5F9;color:#64748B'>sem base</span><span>ano anterior</span>",
        "<span class='pill' style='background:" & IF ( _v >= 0, "rgba(0,203,169,0.14);color:#00A88C'>▲ +", "rgba(232,80,91,0.12);color:#D64550'>▼ -" )
            & FORMAT ( ABS ( _v ), "0.0%", "pt-BR" ) & "</span><span>" & _ano & " vs. " & ( _ano - 1 ) & "</span>"
    )
VAR _ag = [Agendamentos]
VAR _acc = "#2C3E50"
VAR _rgb = "44,62,80"
VAR _valor = IF ( ISBLANK ( [Consultas Realizadas] ), "—", FORMAT ( [Consultas Realizadas], "#,0", "pt-BR" ) ) & "<small>de " & FORMAT ( _ag, "#,0", "pt-BR" ) & " agend.</small>"
VAR _css = "<style>*{box-sizing:border-box;margin:0;padding:0}html,body{background:transparent;overflow:hidden}.card{font-family:Inter,'Segoe UI',Arial,sans-serif;height:calc(100vh - 18px);margin:6px 8px 12px 8px;background:rgba(255,255,255,0.88);backdrop-filter:blur(10px);border:1px solid rgba(226,232,240,0.95);border-radius:16px;padding:14px 18px 12px 22px;box-shadow:0 2px 10px rgba(44,62,80,0.06);transition:transform .35s cubic-bezier(.2,.8,.2,1),box-shadow .35s ease,border-color .35s ease;position:relative;overflow:hidden;display:flex;flex-direction:column;justify-content:space-between;cursor:default}.card:hover{transform:translateY(-4px);box-shadow:0 14px 28px rgba(44,62,80,0.14);border-color:rgba(" & _rgb & ",0.55)}.card:before{content:'';position:absolute;left:0;top:16px;bottom:16px;width:4px;border-radius:0 4px 4px 0;background:" & _acc & "}.card:after{content:'';position:absolute;right:-46px;top:-46px;width:130px;height:130px;border-radius:50%;background:radial-gradient(circle,rgba(" & _rgb & ",0.13),rgba(" & _rgb & ",0) 70%);transition:transform .6s ease}.card:hover:after{transform:scale(1.5)}.top{display:flex;justify-content:space-between;align-items:center;position:relative;z-index:1}.lbl{font-size:11px;font-weight:600;color:#64748B;letter-spacing:.6px;text-transform:uppercase}.ico{width:34px;height:34px;border-radius:10px;background:rgba(" & _rgb & ",0.12);display:flex;align-items:center;justify-content:center;flex-shrink:0}.ico svg{width:19px;height:19px;stroke:" & _acc & ";fill:none;stroke-width:2;stroke-linecap:round;stroke-linejoin:round}.pulse{animation:pulse 2.6s ease-in-out infinite}.spin svg{animation:spin 7s linear infinite}.beat svg polyline{stroke-dasharray:64;stroke-dashoffset:64;animation:draw 2.4s ease-in-out infinite}@keyframes pulse{0%,100%{box-shadow:0 0 0 0 rgba(" & _rgb & ",0.35);transform:scale(1)}50%{box-shadow:0 0 0 7px rgba(" & _rgb & ",0);transform:scale(1.07)}}@keyframes spin{to{transform:rotate(360deg)}}@keyframes draw{0%{stroke-dashoffset:64}50%{stroke-dashoffset:0}100%{stroke-dashoffset:-64}}@keyframes up{from{opacity:0;transform:translateY(10px)}to{opacity:1;transform:none}}@keyframes grow{from{width:0}}.val{font-size:30px;font-weight:700;letter-spacing:-.5px;line-height:1.1;position:relative;z-index:1;animation:up .9s cubic-bezier(.2,.8,.2,1) both}.val small{font-size:11px;font-weight:500;color:#64748B;letter-spacing:0;margin-left:6px}.foot{font-size:11px;color:#64748B;display:flex;gap:8px;align-items:center;white-space:nowrap;overflow:hidden;position:relative;z-index:1}.pill{font-weight:700;padding:2px 8px;border-radius:20px;font-size:10.5px}.bar{flex:1;min-width:60px;height:6px;border-radius:6px;background:#EEF2F6;position:relative;display:flex;overflow:visible}.bar i{display:block;height:100%;border-radius:6px;animation:grow 1.4s cubic-bezier(.2,.8,.2,1) both}.bar b{position:absolute;top:-3px;width:2px;height:12px;border-radius:2px;background:#2C3E50}.stars{position:relative;display:inline-block;color:#E2E8F0;font-size:13px;letter-spacing:2px}.stars span{position:absolute;left:0;top:0;overflow:hidden;white-space:nowrap;color:#F5B700;animation:grow 1.6s ease-out both}.more{position:absolute;inset:0;z-index:5;border-radius:16px;padding:11px 16px 9px 22px;color:#fff;background:linear-gradient(150deg,rgba(44,62,80,0.97) 0%,rgba(31,45,58,0.97) 60%,rgba(27,59,61,0.97) 100%);opacity:0;transform:translateY(14px) scale(.98);transition:opacity .35s ease,transform .45s cubic-bezier(.2,.8,.2,1);pointer-events:none;display:flex;flex-direction:column;justify-content:space-between}.more:before{content:'';position:absolute;left:0;top:16px;bottom:16px;width:4px;border-radius:0 4px 4px 0;background:" & _acc & "}.card:hover .more{opacity:1;transform:none}.mt{font-size:9.5px;font-weight:700;letter-spacing:.9px;text-transform:uppercase;color:#33D6BB;display:flex;justify-content:space-between}.mt em{font-style:normal;color:#94A3B8;font-weight:500;letter-spacing:.3px;text-transform:none}.mg{display:grid;grid-template-columns:1fr 1fr;gap:3px 14px}.mg div{display:flex;justify-content:space-between;align-items:baseline;gap:6px;border-bottom:1px dashed rgba(255,255,255,0.10);padding-bottom:2px;opacity:0;transform:translateY(6px);transition:opacity .4s ease,transform .4s ease}.card:hover .mg div{opacity:1;transform:none}.card:hover .mg div:nth-child(2){transition-delay:.06s}.card:hover .mg div:nth-child(3){transition-delay:.12s}.card:hover .mg div:nth-child(4){transition-delay:.18s}.mg span{font-size:9.5px;color:#94A3B8;white-space:nowrap}.mg b{font-size:12px;font-weight:700;color:#fff;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.mi{font-size:10px;color:#E2E8F0;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;opacity:0;transition:opacity .5s .25s ease}.mi b{color:#33D6BB}.card:hover .mi{opacity:1}.hint{position:absolute;right:10px;bottom:6px;font-size:8.5px;color:#CBD5E1;letter-spacing:.3px;z-index:2;transition:opacity .3s}.card:hover .hint{opacity:0}</style>"
RETURN
    _css
        & "<div class='card'><div class='top'><span class='lbl'>Total de Consultas</span>"
        & "<div class='ico beat'><svg viewBox='0 0 24 24'><polyline points='22 12 18 12 15 21 9 3 6 12 2 12'/></svg></div></div>"
        & "<div class='val' style='color:" & "#2C3E50" & "'>" & _valor & "</div>"
        & "<div class='foot'>" & _pill & "</div>"
        & "<div class='hint'>ⓘ detalhes</div>"
        & "<div class='more'><div class='mt'>Volume de atendimentos</div><div class='mg'>" & "<div><span>Realização</span><b>" & ( IF ( ISBLANK ( [% Realização] ), "—", FORMAT ( [% Realização], "0.0%", "pt-BR" ) ) ) & "</b></div>" & "<div><span>Média/mês</span><b>" & ( IF ( ISBLANK ( [Consultas Média Mensal] ), "—", FORMAT ( [Consultas Média Mensal], "#,0", "pt-BR" ) ) ) & "</b></div>" & "<div><span>Canceladas</span><b>" & ( IF ( ISBLANK ( [Consultas Canceladas] ), "—", FORMAT ( [Consultas Canceladas], "#,0", "pt-BR" ) ) ) & "</b></div>" & "<div><span>No-show</span><b>" & ( IF ( ISBLANK ( [Consultas No-Show] ), "—", FORMAT ( [Consultas No-Show], "#,0", "pt-BR" ) ) ) & "</b></div>" & "</div>"
        & "<div class='mi'>" & ( "Mais procurada: <b>" & [Top Especialidade Consultas] & "</b>" ) & "</div></div>"
        & "</div>"
```

### HTML Card Cancelamento
```dax
HTML Card Cancelamento =
VAR _taxa = [% Cancelamento]
VAR _meta = [Meta Cancelamento]
VAR _ns = [% No-Show]
VAR _acima = _taxa > _meta
VAR _escala = 0.40
VAR _acc = IF ( _acima, "#E8505B", "#00CBA9" )
VAR _rgb = IF ( _acima, "232,80,91", "0,203,169" )
VAR _valor = IF ( ISBLANK ( _taxa ), "—", FORMAT ( _taxa, "0.0%", "pt-BR" ) ) & "<small>+ " & FORMAT ( _ns, "0.0%", "pt-BR" ) & " no-show</small>"
VAR _css = "<style>*{box-sizing:border-box;margin:0;padding:0}html,body{background:transparent;overflow:hidden}.card{font-family:Inter,'Segoe UI',Arial,sans-serif;height:calc(100vh - 18px);margin:6px 8px 12px 8px;background:rgba(255,255,255,0.88);backdrop-filter:blur(10px);border:1px solid rgba(226,232,240,0.95);border-radius:16px;padding:14px 18px 12px 22px;box-shadow:0 2px 10px rgba(44,62,80,0.06);transition:transform .35s cubic-bezier(.2,.8,.2,1),box-shadow .35s ease,border-color .35s ease;position:relative;overflow:hidden;display:flex;flex-direction:column;justify-content:space-between;cursor:default}.card:hover{transform:translateY(-4px);box-shadow:0 14px 28px rgba(44,62,80,0.14);border-color:rgba(" & _rgb & ",0.55)}.card:before{content:'';position:absolute;left:0;top:16px;bottom:16px;width:4px;border-radius:0 4px 4px 0;background:" & _acc & "}.card:after{content:'';position:absolute;right:-46px;top:-46px;width:130px;height:130px;border-radius:50%;background:radial-gradient(circle,rgba(" & _rgb & ",0.13),rgba(" & _rgb & ",0) 70%);transition:transform .6s ease}.card:hover:after{transform:scale(1.5)}.top{display:flex;justify-content:space-between;align-items:center;position:relative;z-index:1}.lbl{font-size:11px;font-weight:600;color:#64748B;letter-spacing:.6px;text-transform:uppercase}.ico{width:34px;height:34px;border-radius:10px;background:rgba(" & _rgb & ",0.12);display:flex;align-items:center;justify-content:center;flex-shrink:0}.ico svg{width:19px;height:19px;stroke:" & _acc & ";fill:none;stroke-width:2;stroke-linecap:round;stroke-linejoin:round}.pulse{animation:pulse 2.6s ease-in-out infinite}.spin svg{animation:spin 7s linear infinite}.beat svg polyline{stroke-dasharray:64;stroke-dashoffset:64;animation:draw 2.4s ease-in-out infinite}@keyframes pulse{0%,100%{box-shadow:0 0 0 0 rgba(" & _rgb & ",0.35);transform:scale(1)}50%{box-shadow:0 0 0 7px rgba(" & _rgb & ",0);transform:scale(1.07)}}@keyframes spin{to{transform:rotate(360deg)}}@keyframes draw{0%{stroke-dashoffset:64}50%{stroke-dashoffset:0}100%{stroke-dashoffset:-64}}@keyframes up{from{opacity:0;transform:translateY(10px)}to{opacity:1;transform:none}}@keyframes grow{from{width:0}}.val{font-size:30px;font-weight:700;letter-spacing:-.5px;line-height:1.1;position:relative;z-index:1;animation:up .9s cubic-bezier(.2,.8,.2,1) both}.val small{font-size:11px;font-weight:500;color:#64748B;letter-spacing:0;margin-left:6px}.foot{font-size:11px;color:#64748B;display:flex;gap:8px;align-items:center;white-space:nowrap;overflow:hidden;position:relative;z-index:1}.pill{font-weight:700;padding:2px 8px;border-radius:20px;font-size:10.5px}.bar{flex:1;min-width:60px;height:6px;border-radius:6px;background:#EEF2F6;position:relative;display:flex;overflow:visible}.bar i{display:block;height:100%;border-radius:6px;animation:grow 1.4s cubic-bezier(.2,.8,.2,1) both}.bar b{position:absolute;top:-3px;width:2px;height:12px;border-radius:2px;background:#2C3E50}.stars{position:relative;display:inline-block;color:#E2E8F0;font-size:13px;letter-spacing:2px}.stars span{position:absolute;left:0;top:0;overflow:hidden;white-space:nowrap;color:#F5B700;animation:grow 1.6s ease-out both}.more{position:absolute;inset:0;z-index:5;border-radius:16px;padding:11px 16px 9px 22px;color:#fff;background:linear-gradient(150deg,rgba(44,62,80,0.97) 0%,rgba(31,45,58,0.97) 60%,rgba(27,59,61,0.97) 100%);opacity:0;transform:translateY(14px) scale(.98);transition:opacity .35s ease,transform .45s cubic-bezier(.2,.8,.2,1);pointer-events:none;display:flex;flex-direction:column;justify-content:space-between}.more:before{content:'';position:absolute;left:0;top:16px;bottom:16px;width:4px;border-radius:0 4px 4px 0;background:" & _acc & "}.card:hover .more{opacity:1;transform:none}.mt{font-size:9.5px;font-weight:700;letter-spacing:.9px;text-transform:uppercase;color:#33D6BB;display:flex;justify-content:space-between}.mt em{font-style:normal;color:#94A3B8;font-weight:500;letter-spacing:.3px;text-transform:none}.mg{display:grid;grid-template-columns:1fr 1fr;gap:3px 14px}.mg div{display:flex;justify-content:space-between;align-items:baseline;gap:6px;border-bottom:1px dashed rgba(255,255,255,0.10);padding-bottom:2px;opacity:0;transform:translateY(6px);transition:opacity .4s ease,transform .4s ease}.card:hover .mg div{opacity:1;transform:none}.card:hover .mg div:nth-child(2){transition-delay:.06s}.card:hover .mg div:nth-child(3){transition-delay:.12s}.card:hover .mg div:nth-child(4){transition-delay:.18s}.mg span{font-size:9.5px;color:#94A3B8;white-space:nowrap}.mg b{font-size:12px;font-weight:700;color:#fff;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.mi{font-size:10px;color:#E2E8F0;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;opacity:0;transition:opacity .5s .25s ease}.mi b{color:#33D6BB}.card:hover .mi{opacity:1}.hint{position:absolute;right:10px;bottom:6px;font-size:8.5px;color:#CBD5E1;letter-spacing:.3px;z-index:2;transition:opacity .3s}.card:hover .hint{opacity:0}</style>"
RETURN
    _css
        & "<div class='card'><div class='top'><span class='lbl'>Taxa de Cancelamento</span>"
        & "<div class='ico pulse'><svg viewBox='0 0 24 24'><circle cx='12' cy='12' r='10'/><path d='M15 9l-6 6M9 9l6 6'/></svg></div></div>"
        & "<div class='val' style='color:" & IF ( _acima, "#D64550", "#00A88C" ) & "'>" & _valor & "</div>"
        & "<div class='foot'>" & "<div class='bar'><i style='width:" & FORMAT ( MIN ( DIVIDE ( _taxa, _escala ), 1 ) * 100, "0.0", "en-US" ) & "%;background:" & _acc & "'></i>"
            & "<b style='left:" & FORMAT ( DIVIDE ( _meta, _escala ) * 100, "0.0", "en-US" ) & "%'></b></div>"
            & "<span>meta &lt; " & FORMAT ( _meta, "0%", "pt-BR" ) & "</span>" & "</div>"
        & "<div class='hint'>ⓘ detalhes</div>"
        & "<div class='more'><div class='mt'>Impacto das perdas</div><div class='mg'>" & "<div><span>Canceladas</span><b>" & ( IF ( ISBLANK ( [Consultas Canceladas] ), "—", FORMAT ( [Consultas Canceladas], "#,0", "pt-BR" ) ) ) & "</b></div>" & "<div><span>No-show</span><b>" & ( IF ( ISBLANK ( [Consultas No-Show] ), "—", FORMAT ( [Consultas No-Show], "#,0", "pt-BR" ) ) ) & "</b></div>" & "<div><span>Perda total</span><b>" & ( IF ( ISBLANK ( [% Perda Total] ), "—", FORMAT ( [% Perda Total], "0.0%", "pt-BR" ) ) ) & "</b></div>" & "<div><span>Receita perdida</span><b>" & ( IF ( ISBLANK ( [Receita Perdida Estimada] ), "—", "R$ " & IF ( [Receita Perdida Estimada] >= 1000000, FORMAT ( [Receita Perdida Estimada] / 1000000, "#,0.0", "pt-BR" ) & " mi", IF ( [Receita Perdida Estimada] >= 1000, FORMAT ( [Receita Perdida Estimada] / 1000, "#,0.0", "pt-BR" ) & " mil", FORMAT ( [Receita Perdida Estimada], "#,0", "pt-BR" ) ) ) ) ) & "</b></div>" & "</div>"
        & "<div class='mi'>" & ( "Maior cancelamento: <b>" & [Top Unidade Cancelamento] & " (" & FORMAT ( CALCULATE ( [% Cancelamento], TREATAS ( { [Top Unidade Cancelamento] }, Dim_Clinicas[Unidade] ) ), "0.0%", "pt-BR" ) & ")" & "</b>" ) & "</div></div>"
        & "</div>"
```

### HTML Card Ticket
```dax
HTML Card Ticket =
VAR _v = [Ticket Médio Var % AA]
VAR _ano = MAX ( Dim_Calendario[Ano] )
VAR _pill =
    IF (
        ISBLANK ( _v ),
        "<span class='pill' style='background:#F1F5F9;color:#64748B'>sem base</span><span>ano anterior</span>",
        "<span class='pill' style='background:" & IF ( _v >= 0, "rgba(0,203,169,0.14);color:#00A88C'>▲ +", "rgba(232,80,91,0.12);color:#D64550'>▼ -" )
            & FORMAT ( ABS ( _v ), "0.0%", "pt-BR" ) & "</span><span>" & _ano & " vs. " & ( _ano - 1 ) & "</span>"
    )
VAR _acc = "#2C3E50"
VAR _rgb = "44,62,80"
VAR _valor = "R$ " & IF ( ISBLANK ( [Ticket Médio] ), "—", FORMAT ( [Ticket Médio], "#,0.00", "pt-BR" ) )
VAR _css = "<style>*{box-sizing:border-box;margin:0;padding:0}html,body{background:transparent;overflow:hidden}.card{font-family:Inter,'Segoe UI',Arial,sans-serif;height:calc(100vh - 18px);margin:6px 8px 12px 8px;background:rgba(255,255,255,0.88);backdrop-filter:blur(10px);border:1px solid rgba(226,232,240,0.95);border-radius:16px;padding:14px 18px 12px 22px;box-shadow:0 2px 10px rgba(44,62,80,0.06);transition:transform .35s cubic-bezier(.2,.8,.2,1),box-shadow .35s ease,border-color .35s ease;position:relative;overflow:hidden;display:flex;flex-direction:column;justify-content:space-between;cursor:default}.card:hover{transform:translateY(-4px);box-shadow:0 14px 28px rgba(44,62,80,0.14);border-color:rgba(" & _rgb & ",0.55)}.card:before{content:'';position:absolute;left:0;top:16px;bottom:16px;width:4px;border-radius:0 4px 4px 0;background:" & _acc & "}.card:after{content:'';position:absolute;right:-46px;top:-46px;width:130px;height:130px;border-radius:50%;background:radial-gradient(circle,rgba(" & _rgb & ",0.13),rgba(" & _rgb & ",0) 70%);transition:transform .6s ease}.card:hover:after{transform:scale(1.5)}.top{display:flex;justify-content:space-between;align-items:center;position:relative;z-index:1}.lbl{font-size:11px;font-weight:600;color:#64748B;letter-spacing:.6px;text-transform:uppercase}.ico{width:34px;height:34px;border-radius:10px;background:rgba(" & _rgb & ",0.12);display:flex;align-items:center;justify-content:center;flex-shrink:0}.ico svg{width:19px;height:19px;stroke:" & _acc & ";fill:none;stroke-width:2;stroke-linecap:round;stroke-linejoin:round}.pulse{animation:pulse 2.6s ease-in-out infinite}.spin svg{animation:spin 7s linear infinite}.beat svg polyline{stroke-dasharray:64;stroke-dashoffset:64;animation:draw 2.4s ease-in-out infinite}@keyframes pulse{0%,100%{box-shadow:0 0 0 0 rgba(" & _rgb & ",0.35);transform:scale(1)}50%{box-shadow:0 0 0 7px rgba(" & _rgb & ",0);transform:scale(1.07)}}@keyframes spin{to{transform:rotate(360deg)}}@keyframes draw{0%{stroke-dashoffset:64}50%{stroke-dashoffset:0}100%{stroke-dashoffset:-64}}@keyframes up{from{opacity:0;transform:translateY(10px)}to{opacity:1;transform:none}}@keyframes grow{from{width:0}}.val{font-size:30px;font-weight:700;letter-spacing:-.5px;line-height:1.1;position:relative;z-index:1;animation:up .9s cubic-bezier(.2,.8,.2,1) both}.val small{font-size:11px;font-weight:500;color:#64748B;letter-spacing:0;margin-left:6px}.foot{font-size:11px;color:#64748B;display:flex;gap:8px;align-items:center;white-space:nowrap;overflow:hidden;position:relative;z-index:1}.pill{font-weight:700;padding:2px 8px;border-radius:20px;font-size:10.5px}.bar{flex:1;min-width:60px;height:6px;border-radius:6px;background:#EEF2F6;position:relative;display:flex;overflow:visible}.bar i{display:block;height:100%;border-radius:6px;animation:grow 1.4s cubic-bezier(.2,.8,.2,1) both}.bar b{position:absolute;top:-3px;width:2px;height:12px;border-radius:2px;background:#2C3E50}.stars{position:relative;display:inline-block;color:#E2E8F0;font-size:13px;letter-spacing:2px}.stars span{position:absolute;left:0;top:0;overflow:hidden;white-space:nowrap;color:#F5B700;animation:grow 1.6s ease-out both}.more{position:absolute;inset:0;z-index:5;border-radius:16px;padding:11px 16px 9px 22px;color:#fff;background:linear-gradient(150deg,rgba(44,62,80,0.97) 0%,rgba(31,45,58,0.97) 60%,rgba(27,59,61,0.97) 100%);opacity:0;transform:translateY(14px) scale(.98);transition:opacity .35s ease,transform .45s cubic-bezier(.2,.8,.2,1);pointer-events:none;display:flex;flex-direction:column;justify-content:space-between}.more:before{content:'';position:absolute;left:0;top:16px;bottom:16px;width:4px;border-radius:0 4px 4px 0;background:" & _acc & "}.card:hover .more{opacity:1;transform:none}.mt{font-size:9.5px;font-weight:700;letter-spacing:.9px;text-transform:uppercase;color:#33D6BB;display:flex;justify-content:space-between}.mt em{font-style:normal;color:#94A3B8;font-weight:500;letter-spacing:.3px;text-transform:none}.mg{display:grid;grid-template-columns:1fr 1fr;gap:3px 14px}.mg div{display:flex;justify-content:space-between;align-items:baseline;gap:6px;border-bottom:1px dashed rgba(255,255,255,0.10);padding-bottom:2px;opacity:0;transform:translateY(6px);transition:opacity .4s ease,transform .4s ease}.card:hover .mg div{opacity:1;transform:none}.card:hover .mg div:nth-child(2){transition-delay:.06s}.card:hover .mg div:nth-child(3){transition-delay:.12s}.card:hover .mg div:nth-child(4){transition-delay:.18s}.mg span{font-size:9.5px;color:#94A3B8;white-space:nowrap}.mg b{font-size:12px;font-weight:700;color:#fff;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.mi{font-size:10px;color:#E2E8F0;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;opacity:0;transition:opacity .5s .25s ease}.mi b{color:#33D6BB}.card:hover .mi{opacity:1}.hint{position:absolute;right:10px;bottom:6px;font-size:8.5px;color:#CBD5E1;letter-spacing:.3px;z-index:2;transition:opacity .3s}.card:hover .hint{opacity:0}</style>"
RETURN
    _css
        & "<div class='card'><div class='top'><span class='lbl'>Ticket Médio</span>"
        & "<div class='ico spin'><svg viewBox='0 0 24 24'><path d='M20.59 13.41l-7.17 7.17a2 2 0 0 1-2.83 0L2 12V2h10l8.59 8.59a2 2 0 0 1 0 2.82z'/><line x1='7' y1='7' x2='7.01' y2='7'/></svg></div></div>"
        & "<div class='val' style='color:" & "#2C3E50" & "'>" & _valor & "</div>"
        & "<div class='foot'>" & _pill & "</div>"
        & "<div class='hint'>ⓘ detalhes</div>"
        & "<div class='more'><div class='mt'>Composição do ticket</div><div class='mg'>" & "<div><span>Particular</span><b>" & ( "R$ " & IF ( ISBLANK ( [Ticket Médio Particular] ), "—", FORMAT ( [Ticket Médio Particular], "#,0", "pt-BR" ) ) ) & "</b></div>" & "<div><span>Convênio</span><b>" & ( "R$ " & IF ( ISBLANK ( [Ticket Médio Convênio] ), "—", FORMAT ( [Ticket Médio Convênio], "#,0", "pt-BR" ) ) ) & "</b></div>" & "<div><span>Preço tabela</span><b>" & ( "R$ " & IF ( ISBLANK ( [Valor Tabela Médio] ), "—", FORMAT ( [Valor Tabela Médio], "#,0", "pt-BR" ) ) ) & "</b></div>" & "<div><span>Realização</span><b>" & ( IF ( ISBLANK ( [% Realização de Preço] ), "—", FORMAT ( [% Realização de Preço], "0.0%", "pt-BR" ) ) ) & "</b></div>" & "</div>"
        & "<div class='mi'>" & ( "Maior ticket: <b>" & [Top Especialidade Ticket] & " (" & "R$ " & FORMAT ( CALCULATE ( [Ticket Médio], TREATAS ( { [Top Especialidade Ticket] }, Dim_Especialidades[Especialidade] ) ), "#,0", "pt-BR" ) & ")" & "</b>" ) & "</div></div>"
        & "</div>"
```

### HTML Hero Executivo
```dax
HTML Hero Executivo =
VAR _ini = MIN ( Dim_Calendario[Data] )
VAR _fim = MAX ( Dim_Calendario[Data] )
VAR _periodo = FORMAT ( _ini, "mmm/yy", "pt-BR" ) & " – " & FORMAT ( _fim, "mmm/yy", "pt-BR" )
VAR _unid = COUNTROWS ( VALUES ( Dim_Clinicas[Unidade] ) )
VAR _hora = HOUR ( UTCNOW () - 3 / 24 )  -- horário de Brasília
VAR _saud = IF ( _hora < 12, "Bom dia", IF ( _hora < 18, "Boa tarde", "Boa noite" ) )
RETURN
    "<style>*{box-sizing:border-box;margin:0;padding:0}html,body{background:transparent;overflow:hidden}"
        & ".hero{font-family:Inter,'Segoe UI',Arial,sans-serif;height:calc(100vh - 16px);margin:6px 8px 10px 8px;border-radius:18px;padding:0 28px;"
        & "background:linear-gradient(120deg,#2C3E50 0%,#22313F 55%,#1B4A4A 100%);color:#fff;display:flex;justify-content:space-between;align-items:center;"
        & "position:relative;overflow:hidden;box-shadow:0 10px 26px rgba(44,62,80,0.18);transition:box-shadow .4s ease}"
        & ".hero:hover{box-shadow:0 16px 34px rgba(44,62,80,0.26)}"
        & ".hero:before{content:'';position:absolute;top:0;left:-40%;width:35%;height:100%;background:linear-gradient(100deg,rgba(255,255,255,0) 0%,rgba(255,255,255,0.08) 50%,rgba(255,255,255,0) 100%);animation:shine 6s ease-in-out infinite}"
        & ".hero:after{content:'';position:absolute;right:220px;top:-60px;width:180px;height:180px;border-radius:50%;background:radial-gradient(circle,rgba(0,203,169,0.28),rgba(0,203,169,0) 70%);animation:float 7s ease-in-out infinite}"
        & "@keyframes shine{0%{left:-40%}60%,100%{left:120%}}"
        & "@keyframes float{0%,100%{transform:translateY(0)}50%{transform:translateY(18px)}}"
        & "@keyframes blink{0%,100%{box-shadow:0 0 0 0 rgba(46,213,115,0.6)}50%{box-shadow:0 0 0 6px rgba(46,213,115,0)}}"
        & "@keyframes up{from{opacity:0;transform:translateY(8px)}to{opacity:1;transform:none}}"
        & ".t{font-size:24px;font-weight:700;letter-spacing:-.4px;animation:up .8s ease both}.t span{color:#00CBA9}"
        & ".s{font-size:12.5px;color:#CBD5E1;margin-top:6px;font-weight:400;animation:up .8s .15s ease both}.s b{color:#fff;font-weight:600}"
        & ".st{position:relative;z-index:1;background:rgba(255,255,255,0.08);border:1px solid rgba(0,203,169,0.35);border-radius:10px;padding:8px 16px;backdrop-filter:blur(6px)}"
        & ".sl{font-size:10px;color:#00CBA9;font-weight:700;letter-spacing:.8px;text-transform:uppercase}"
        & ".sv{font-size:13px;font-weight:500;margin-top:3px;display:flex;align-items:center;gap:8px}"
        & ".dot{width:9px;height:9px;border-radius:50%;background:#2ED573;animation:blink 1.8s infinite}</style>"
        & "<div class='hero'><div style='position:relative;z-index:1'>"
        & "<div class='t'>" & _saud & ". Portal Executivo <span>VittaCare</span></div>"
        & "<div class='s'>Faturamento, volume de atendimentos e cancelamentos · <b>" & _periodo & "</b> · <b>" & _unid & "</b> unidade(s)</div></div>"
        & "<div class='st'><div class='sl'>Status do Sistema</div><div class='sv'><i class='dot'></i>Dados atualizados</div></div></div>"
```

Outros cartões no modelo: `HTML Card Nota`, `HTML Card Espera`, `HTML Card Realizadas`, `HTML Card Pacientes`, `HTML Card Agendamentos`, `HTML Card Convenios`, `HTML Card Particular`, `HTML Card NoShow`.

**Configuração do visual:** campo *Values* = medida HTML · Fundo, Borda, Sombra e Título do contentor **desligados** · cabeçalho do visual oculto.

## 6. Deneb (Vega-Lite)

Em todos: *Provider* = Vega-Lite, *Render* = SVG, *Tooltips* ligadas. Os campos entram em **Values** com os nomes abaixo.

### Config (separador *Config* do Deneb, igual para todos)
```json
{
  "autosize": {
    "type": "fit",
    "contains": "padding"
  },
  "padding": {
    "left": 4,
    "right": 8,
    "top": 8,
    "bottom": 4
  },
  "background": null,
  "font": "Inter, Segoe UI, Arial, sans-serif",
  "view": {
    "stroke": "transparent"
  },
  "axis": {
    "domain": false,
    "ticks": false,
    "grid": false,
    "labelColor": "#64748B",
    "labelFontSize": 10,
    "labelFont": "Inter, Segoe UI, Arial, sans-serif",
    "titleColor": "#64748B",
    "titleFontSize": 10,
    "titleFontWeight": "normal",
    "labelPadding": 6
  },
  "legend": {
    "labelColor": "#64748B",
    "labelFontSize": 10,
    "labelFont": "Inter, Segoe UI, Arial, sans-serif",
    "titleColor": "#64748B",
    "symbolType": "circle",
    "symbolSize": 80
  },
  "text": {
    "font": "Inter, Segoe UI, Arial, sans-serif",
    "color": "#2C3E50"
  },
  "line": {
    "strokeCap": "round"
  },
  "range": {
    "category": [
      "#00CBA9",
      "#2C3E50",
      "#33D6BB",
      "#5D6D7E",
      "#80E5D4",
      "#AAB7C4"
    ]
  }
}
```

### Barras arredondadas — Faturamento por Unidade
**Campos:** Dim_Clinicas[Unidade], [Faturamento Total], [Consultas Realizadas], [Ticket Médio]
```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {
    "name": "dataset"
  },
  "transform": [
    {
      "joinaggregate": [
        {
          "op": "sum",
          "field": "Faturamento Total",
          "as": "total"
        },
        {
          "op": "max",
          "field": "Faturamento Total",
          "as": "maximo"
        }
      ]
    },
    {
      "calculate": "datum['Faturamento Total'] / datum.total",
      "as": "pct"
    },
    {
      "calculate": "datum.maximo * 1.32",
      "as": "trilho"
    },
    {
      "window": [
        {
          "op": "rank",
          "as": "rank"
        }
      ],
      "sort": [
        {
          "field": "Faturamento Total",
          "order": "descending"
        }
      ]
    }
  ],
  "encoding": {
    "y": {
      "field": "Unidade",
      "type": "nominal",
      "sort": {
        "field": "Faturamento Total",
        "order": "descending"
      },
      "axis": {
        "title": null,
        "labelFontSize": 11,
        "labelColor": "#2C3E50",
        "labelPadding": 8
      }
    }
  },
  "layer": [
    {
      "mark": {
        "type": "bar",
        "color": "#F1F5F9",
        "cornerRadius": 7,
        "height": {
          "band": 0.62
        }
      },
      "encoding": {
        "x": {
          "field": "trilho",
          "type": "quantitative",
          "axis": null,
          "scale": {
            "zero": true
          }
        }
      }
    },
    {
      "mark": {
        "type": "bar",
        "cornerRadiusEnd": 7,
        "height": {
          "band": 0.62
        },
        "cursor": "pointer",
        "tooltip": true
      },
      "encoding": {
        "x": {
          "field": "Faturamento Total",
          "type": "quantitative"
        },
        "color": {
          "condition": {
            "test": "datum.rank == 1",
            "value": "#00CBA9"
          },
          "value": "#2C3E50"
        },
        "opacity": {
          "condition": {
            "test": "datum.rank == 1",
            "value": 1
          },
          "value": 0.82
        }
      }
    },
    {
      "mark": {
        "type": "text",
        "align": "left",
        "dx": 8,
        "fontSize": 11,
        "fontWeight": 600,
        "color": "#2C3E50"
      },
      "encoding": {
        "x": {
          "field": "Faturamento Total",
          "type": "quantitative"
        },
        "text": {
          "field": "Faturamento Total",
          "formatType": "pbiFormat",
          "format": "R$ #,0"
        }
      }
    },
    {
      "mark": {
        "type": "text",
        "align": "right",
        "dx": -4,
        "fontSize": 10,
        "color": "#64748B"
      },
      "encoding": {
        "x": {
          "field": "trilho",
          "type": "quantitative"
        },
        "text": {
          "field": "pct",
          "format": ".0%"
        }
      }
    }
  ]
}
```

### Lollipop — Ranking de Médicos por Nota
**Campos:** Dim_Medicos[Médico], Dim_Especialidades[Especialidade], [Nota Média], [Consultas Realizadas], [Tempo Espera Médio], [Faturamento Total]
```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {
    "name": "dataset"
  },
  "transform": [
    {
      "filter": "isValid(datum['Nota Média'])"
    },
    {
      "window": [
        {
          "op": "row_number",
          "as": "rank"
        }
      ],
      "sort": [
        {
          "field": "Nota Média",
          "order": "descending"
        }
      ]
    },
    {
      "calculate": "3",
      "as": "base"
    }
  ],
  "encoding": {
    "y": {
      "field": "Médico",
      "type": "nominal",
      "sort": {
        "field": "Nota Média",
        "order": "descending"
      },
      "axis": {
        "title": null,
        "labelColor": "#2C3E50",
        "labelFontSize": 10
      }
    }
  },
  "layer": [
    {
      "mark": {
        "type": "rule",
        "strokeWidth": 2.5,
        "strokeCap": "round"
      },
      "encoding": {
        "x": {
          "field": "base",
          "type": "quantitative",
          "scale": {
            "domain": [
              3,
              5
            ],
            "nice": false
          },
          "axis": {
            "title": null,
            "grid": true,
            "gridColor": "#EEF2F6",
            "gridDash": [
              3,
              3
            ],
            "values": [
              3,
              3.5,
              4,
              4.5,
              5
            ],
            "format": ".1f"
          }
        },
        "x2": {
          "field": "Nota Média"
        },
        "color": {
          "condition": {
            "test": "datum.rank == 1",
            "value": "#00CBA9"
          },
          "value": "#CBD5E1"
        }
      }
    },
    {
      "mark": {
        "type": "circle",
        "size": 190,
        "opacity": 1,
        "stroke": "white",
        "strokeWidth": 2,
        "cursor": "pointer",
        "tooltip": true
      },
      "encoding": {
        "x": {
          "field": "Nota Média",
          "type": "quantitative"
        },
        "color": {
          "condition": {
            "test": "datum.rank == 1",
            "value": "#00CBA9"
          },
          "value": "#2C3E50"
        }
      }
    },
    {
      "mark": {
        "type": "text",
        "align": "left",
        "dx": 12,
        "fontSize": 10,
        "fontWeight": 600,
        "color": "#2C3E50"
      },
      "encoding": {
        "x": {
          "field": "Nota Média",
          "type": "quantitative"
        },
        "text": {
          "field": "Nota Média",
          "formatType": "pbiFormat",
          "format": "0.00"
        }
      }
    }
  ]
}
```

### Scatter — Tempo de Espera × Nota
**Campos:** Fato_Atendimentos[ID_Atendimento], Dim_Medicos[Médico], Dim_Especialidades[Especialidade], [Tempo Espera Médio], [Nota Média]
```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {
    "name": "dataset"
  },
  "transform": [
    {
      "filter": "isValid(datum['Nota Média']) && isValid(datum['Tempo Espera Médio'])"
    },
    {
      "calculate": "datum['Nota Média'] + (((datum.ID_Atendimento * 9301 + 49297) % 233280) / 233280 - 0.5) * 0.6",
      "as": "notaJitter"
    },
    {
      "calculate": "datum['Tempo Espera Médio'] > 30 ? 'Acima da meta (> 30 min)' : 'Dentro da meta (≤ 30 min)'",
      "as": "faixa"
    }
  ],
  "layer": [
    {
      "data": {
        "values": [
          {}
        ]
      },
      "mark": {
        "type": "rect",
        "color": "#E8505B",
        "opacity": 0.05
      },
      "encoding": {
        "x": {
          "datum": 30,
          "type": "quantitative"
        },
        "x2": {
          "datum": 62
        }
      }
    },
    {
      "data": {
        "values": [
          {}
        ]
      },
      "mark": {
        "type": "rule",
        "color": "#E8505B",
        "strokeDash": [
          4,
          4
        ],
        "strokeWidth": 1.2,
        "opacity": 0.7
      },
      "encoding": {
        "x": {
          "datum": 30,
          "type": "quantitative"
        }
      }
    },
    {
      "data": {
        "values": [
          {}
        ]
      },
      "mark": {
        "type": "text",
        "color": "#E8505B",
        "align": "left",
        "baseline": "top",
        "dx": 5,
        "y": 4,
        "fontSize": 10,
        "fontWeight": 600,
        "text": "Meta 30 min"
      },
      "encoding": {
        "x": {
          "datum": 30,
          "type": "quantitative"
        }
      }
    },
    {
      "params": [
        {
          "name": "legenda",
          "select": {
            "type": "point",
            "fields": [
              "faixa"
            ]
          },
          "bind": "legend"
        }
      ],
      "mark": {
        "type": "circle",
        "size": 52,
        "stroke": "white",
        "strokeWidth": 0.6,
        "cursor": "pointer",
        "tooltip": true
      },
      "encoding": {
        "x": {
          "field": "Tempo Espera Médio",
          "type": "quantitative",
          "title": "Tempo de espera (min)",
          "scale": {
            "domain": [
              0,
              62
            ],
            "nice": false
          },
          "axis": {
            "grid": true,
            "gridColor": "#EEF2F6",
            "gridDash": [
              3,
              3
            ],
            "values": [
              0,
              10,
              20,
              30,
              40,
              50,
              60
            ]
          }
        },
        "y": {
          "field": "notaJitter",
          "type": "quantitative",
          "title": "Nota do atendimento",
          "scale": {
            "domain": [
              2.5,
              5.5
            ],
            "nice": false
          },
          "axis": {
            "values": [
              3,
              4,
              5
            ],
            "grid": true,
            "gridColor": "#EEF2F6",
            "gridDash": [
              3,
              3
            ],
            "format": ".0f"
          }
        },
        "color": {
          "field": "faixa",
          "type": "nominal",
          "scale": {
            "domain": [
              "Dentro da meta (≤ 30 min)",
              "Acima da meta (> 30 min)"
            ],
            "range": [
              "#00CBA9",
              "#2C3E50"
            ]
          },
          "legend": {
            "orient": "top",
            "title": null,
            "direction": "horizontal"
          }
        },
        "opacity": {
          "condition": {
            "param": "legenda",
            "value": 0.6
          },
          "value": 0.08
        }
      }
    },
    {
      "transform": [
        {
          "regression": "Nota Média",
          "on": "Tempo Espera Médio",
          "extent": [
            0,
            62
          ]
        }
      ],
      "mark": {
        "type": "line",
        "color": "#2C3E50",
        "strokeWidth": 2.2,
        "strokeDash": [
          7,
          5
        ]
      },
      "encoding": {
        "x": {
          "field": "Tempo Espera Médio",
          "type": "quantitative"
        },
        "y": {
          "field": "Nota Média",
          "type": "quantitative"
        }
      }
    }
  ]
}
```

### Área com gradiente — Evolução do Faturamento
**Campos:** Dim_Calendario[AnoMes], Dim_Calendario[Mês Ano], [Faturamento Total], [Consultas Realizadas], [Ticket Médio]
```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {
    "name": "dataset"
  },
  "transform": [
    {
      "joinaggregate": [
        {
          "op": "mean",
          "field": "Faturamento Total",
          "as": "media"
        },
        {
          "op": "max",
          "field": "Faturamento Total",
          "as": "maximo"
        }
      ]
    }
  ],
  "layer": [
    {
      "mark": {
        "type": "area",
        "interpolate": "monotone",
        "color": {
          "gradient": "linear",
          "x1": 1,
          "y1": 1,
          "x2": 1,
          "y2": 0,
          "stops": [
            {
              "offset": 0,
              "color": "rgba(0,203,169,0.02)"
            },
            {
              "offset": 1,
              "color": "rgba(0,203,169,0.38)"
            }
          ]
        }
      },
      "encoding": {
        "x": {
          "field": "Mês Ano",
          "type": "ordinal",
          "sort": {
            "field": "AnoMes",
            "op": "min",
            "order": "ascending"
          },
          "axis": {
            "title": null,
            "labelAngle": 0,
            "labelOverlap": "parity",
            "labelFontSize": 9
          }
        },
        "y": {
          "field": "Faturamento Total",
          "type": "quantitative",
          "axis": {
            "title": null,
            "grid": true,
            "gridColor": "#EEF2F6",
            "gridDash": [
              3,
              3
            ],
            "tickCount": 4,
            "formatType": "pbiFormat",
            "format": "R$ #,0"
          },
          "scale": {
            "zero": true
          }
        }
      }
    },
    {
      "mark": {
        "type": "line",
        "interpolate": "monotone",
        "color": "#00CBA9",
        "strokeWidth": 2.6
      },
      "encoding": {
        "x": {
          "field": "Mês Ano",
          "type": "ordinal",
          "sort": {
            "field": "AnoMes",
            "op": "min",
            "order": "ascending"
          },
          "axis": {
            "title": null,
            "labelAngle": 0,
            "labelOverlap": "parity",
            "labelFontSize": 9
          }
        },
        "y": {
          "field": "Faturamento Total",
          "type": "quantitative"
        }
      }
    },
    {
      "transform": [
        {
          "filter": "datum.AnoMes % 100 == 1"
        }
      ],
      "mark": {
        "type": "rule",
        "color": "#CBD5E1",
        "strokeWidth": 1
      },
      "encoding": {
        "x": {
          "field": "Mês Ano",
          "type": "ordinal",
          "sort": {
            "field": "AnoMes",
            "op": "min",
            "order": "ascending"
          },
          "axis": {
            "title": null,
            "labelAngle": 0,
            "labelOverlap": "parity",
            "labelFontSize": 9
          }
        }
      }
    },
    {
      "transform": [
        {
          "filter": "datum.AnoMes % 100 == 1"
        },
        {
          "calculate": "toString(floor(datum.AnoMes / 100))",
          "as": "anoTxt"
        }
      ],
      "mark": {
        "type": "text",
        "align": "left",
        "baseline": "top",
        "dx": 5,
        "y": 2,
        "fontSize": 11,
        "fontWeight": 700,
        "color": "#94A3B8"
      },
      "encoding": {
        "x": {
          "field": "Mês Ano",
          "type": "ordinal",
          "sort": {
            "field": "AnoMes",
            "op": "min",
            "order": "ascending"
          },
          "axis": {
            "title": null,
            "labelAngle": 0,
            "labelOverlap": "parity",
            "labelFontSize": 9
          }
        },
        "text": {
          "field": "anoTxt"
        }
      }
    },
    {
      "mark": {
        "type": "rule",
        "color": "#2C3E50",
        "strokeDash": [
          4,
          4
        ],
        "strokeWidth": 1,
        "opacity": 0.55
      },
      "encoding": {
        "y": {
          "field": "media",
          "aggregate": "min",
          "type": "quantitative"
        }
      }
    },
    {
      "mark": {
        "type": "text",
        "align": "right",
        "baseline": "bottom",
        "dy": -4,
        "fontSize": 10,
        "color": "#2C3E50",
        "opacity": 0.75,
        "x": {
          "expr": "width"
        }
      },
      "encoding": {
        "y": {
          "field": "media",
          "aggregate": "min",
          "type": "quantitative"
        },
        "text": {
          "field": "media",
          "aggregate": "min",
          "type": "quantitative",
          "formatType": "pbiFormat",
          "format": "R$ #,0"
        }
      }
    },
    {
      "mark": {
        "type": "rule",
        "color": "#94A3B8",
        "strokeWidth": 1
      },
      "encoding": {
        "x": {
          "field": "Mês Ano",
          "type": "ordinal",
          "sort": {
            "field": "AnoMes",
            "op": "min",
            "order": "ascending"
          },
          "axis": {
            "title": null,
            "labelAngle": 0,
            "labelOverlap": "parity",
            "labelFontSize": 9
          }
        },
        "opacity": {
          "condition": {
            "param": "hover",
            "empty": false,
            "value": 0.7
          },
          "value": 0
        }
      }
    },
    {
      "mark": {
        "type": "point",
        "filled": true,
        "size": 80,
        "color": "#00CBA9",
        "stroke": "white",
        "strokeWidth": 2
      },
      "encoding": {
        "x": {
          "field": "Mês Ano",
          "type": "ordinal",
          "sort": {
            "field": "AnoMes",
            "op": "min",
            "order": "ascending"
          },
          "axis": {
            "title": null,
            "labelAngle": 0,
            "labelOverlap": "parity",
            "labelFontSize": 9
          }
        },
        "y": {
          "field": "Faturamento Total",
          "type": "quantitative"
        },
        "opacity": {
          "condition": {
            "param": "hover",
            "empty": false,
            "value": 1
          },
          "value": 0
        }
      }
    },
    {
      "params": [
        {
          "name": "hover",
          "select": {
            "type": "point",
            "encodings": [
              "x"
            ],
            "on": "mouseover",
            "clear": "mouseout"
          }
        }
      ],
      "mark": {
        "type": "bar",
        "opacity": 0,
        "width": {
          "band": 1
        },
        "cursor": "pointer",
        "tooltip": true
      },
      "encoding": {
        "x": {
          "field": "Mês Ano",
          "type": "ordinal",
          "sort": {
            "field": "AnoMes",
            "op": "min",
            "order": "ascending"
          },
          "axis": {
            "title": null,
            "labelAngle": 0,
            "labelOverlap": "parity",
            "labelFontSize": 9
          }
        },
        "y": {
          "field": "maximo",
          "type": "quantitative"
        },
        "y2": {
          "datum": 0
        }
      }
    }
  ]
}
```

### Donut — Consultas por Especialidade
**Campos:** Dim_Especialidades[Especialidade], [Consultas Realizadas], [Faturamento Total]
```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {
    "name": "dataset"
  },
  "transform": [
    {
      "joinaggregate": [
        {
          "op": "sum",
          "field": "Consultas Realizadas",
          "as": "tot"
        }
      ]
    },
    {
      "calculate": "datum['Consultas Realizadas'] / datum.tot",
      "as": "pct"
    },
    {
      "calculate": "datum.Especialidade + '  ' + format(datum.pct, '.0%')",
      "as": "rotulo"
    }
  ],
  "layer": [
    {
      "mark": {
        "type": "arc",
        "innerRadius": {
          "expr": "min(width, height) / 2 * 0.64"
        },
        "outerRadius": {
          "expr": "min(width, height) / 2 - 2"
        },
        "padAngle": 0.025,
        "cornerRadius": 5,
        "cursor": "pointer",
        "tooltip": true
      },
      "encoding": {
        "theta": {
          "field": "Consultas Realizadas",
          "type": "quantitative",
          "stack": true
        },
        "color": {
          "field": "rotulo",
          "type": "nominal",
          "sort": {
            "field": "Consultas Realizadas",
            "order": "descending"
          },
          "scale": {
            "range": [
              "#00CBA9",
              "#2C3E50",
              "#33D6BB",
              "#5D6D7E",
              "#80E5D4",
              "#AAB7C4"
            ]
          },
          "legend": {
            "orient": "right",
            "title": null,
            "labelLimit": 160,
            "rowPadding": 6
          }
        },
        "order": {
          "field": "Consultas Realizadas",
          "type": "quantitative",
          "sort": "descending"
        }
      }
    },
    {
      "mark": {
        "type": "text",
        "fontSize": 22,
        "fontWeight": 700,
        "color": "#2C3E50",
        "dy": -7
      },
      "encoding": {
        "text": {
          "aggregate": "sum",
          "field": "Consultas Realizadas",
          "type": "quantitative",
          "formatType": "pbiFormat",
          "format": "#,0"
        }
      }
    },
    {
      "mark": {
        "type": "text",
        "fontSize": 10,
        "color": "#64748B",
        "dy": 13,
        "text": "consultas"
      }
    }
  ]
}
```

### Colunas arredondadas — Receita por Especialidade
**Campos:** Dim_Especialidades[Especialidade], [Faturamento Total], [Consultas Realizadas], [Ticket Médio], [Nota Média]
```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {
    "name": "dataset"
  },
  "transform": [
    {
      "window": [
        {
          "op": "rank",
          "as": "rank"
        }
      ],
      "sort": [
        {
          "field": "Faturamento Total",
          "order": "descending"
        }
      ]
    },
    {
      "calculate": "'ticket R$ ' + format(datum['Ticket Médio'], ',.0f')",
      "as": "tkt"
    }
  ],
  "encoding": {
    "x": {
      "field": "Especialidade",
      "type": "nominal",
      "sort": {
        "field": "Faturamento Total",
        "order": "descending"
      },
      "axis": {
        "title": null,
        "labelAngle": 0,
        "labelLimit": 95,
        "labelColor": "#2C3E50"
      }
    }
  },
  "layer": [
    {
      "mark": {
        "type": "bar",
        "cornerRadiusTopLeft": 8,
        "cornerRadiusTopRight": 8,
        "width": {
          "band": 0.58
        },
        "cursor": "pointer",
        "tooltip": true
      },
      "encoding": {
        "y": {
          "field": "Faturamento Total",
          "type": "quantitative",
          "axis": null,
          "scale": {
            "zero": true
          }
        },
        "color": {
          "condition": {
            "test": "datum.rank == 1",
            "value": "#00CBA9"
          },
          "value": "#2C3E50"
        },
        "opacity": {
          "condition": {
            "test": "datum.rank == 1",
            "value": 1
          },
          "value": 0.82
        }
      }
    },
    {
      "mark": {
        "type": "text",
        "baseline": "bottom",
        "dy": -16,
        "fontSize": 11,
        "fontWeight": 700,
        "color": "#2C3E50"
      },
      "encoding": {
        "y": {
          "field": "Faturamento Total",
          "type": "quantitative"
        },
        "text": {
          "field": "Faturamento Total",
          "formatType": "pbiFormat",
          "format": "R$ #,0"
        }
      }
    },
    {
      "mark": {
        "type": "text",
        "baseline": "bottom",
        "dy": -4,
        "fontSize": 9,
        "color": "#64748B"
      },
      "encoding": {
        "y": {
          "field": "Faturamento Total",
          "type": "quantitative"
        },
        "text": {
          "field": "tkt"
        }
      }
    }
  ]
}
```

### Barras 100% — Status por Plano de Saúde
**Campos:** Dim_Pacientes[Plano de Saúde], Fato_Atendimentos[Status Consulta], [Agendamentos]
```json
{
  "$schema": "https://vega.github.io/schema/vega-lite/v5.json",
  "data": {
    "name": "dataset"
  },
  "transform": [
    {
      "joinaggregate": [
        {
          "op": "sum",
          "field": "Agendamentos",
          "as": "totPlano"
        }
      ],
      "groupby": [
        "Plano de Saúde"
      ]
    },
    {
      "calculate": "datum.Agendamentos / datum.totPlano",
      "as": "pct"
    },
    {
      "calculate": "indexof(['Realizada','Cancelada','No-Show'], datum['Status Consulta'])",
      "as": "ordem"
    },
    {
      "window": [
        {
          "op": "sum",
          "field": "pct",
          "as": "acum"
        }
      ],
      "groupby": [
        "Plano de Saúde"
      ],
      "sort": [
        {
          "field": "ordem",
          "order": "ascending"
        }
      ],
      "frame": [
        null,
        0
      ]
    },
    {
      "calculate": "datum.acum - datum.pct / 2",
      "as": "meio"
    }
  ],
  "encoding": {
    "y": {
      "field": "Plano de Saúde",
      "type": "nominal",
      "sort": {
        "field": "totPlano",
        "op": "max",
        "order": "descending"
      },
      "axis": {
        "title": null,
        "labelColor": "#2C3E50",
        "labelFontSize": 11
      }
    }
  },
  "layer": [
    {
      "mark": {
        "type": "bar",
        "cornerRadius": 4,
        "stroke": "white",
        "strokeWidth": 2,
        "height": {
          "band": 0.62
        },
        "cursor": "pointer",
        "tooltip": true
      },
      "encoding": {
        "x": {
          "field": "Agendamentos",
          "type": "quantitative",
          "stack": "normalize",
          "axis": null,
          "scale": {
            "domain": [
              0,
              1
            ]
          }
        },
        "color": {
          "field": "Status Consulta",
          "type": "nominal",
          "scale": {
            "domain": [
              "Realizada",
              "Cancelada",
              "No-Show"
            ],
            "range": [
              "#00CBA9",
              "#F28B95",
              "#2C3E50"
            ]
          },
          "legend": {
            "orient": "top",
            "title": null,
            "direction": "horizontal"
          }
        },
        "order": {
          "field": "ordem",
          "type": "quantitative"
        }
      }
    },
    {
      "transform": [
        {
          "filter": "datum.pct >= 0.09"
        }
      ],
      "mark": {
        "type": "text",
        "color": "white",
        "fontSize": 10,
        "fontWeight": 600
      },
      "encoding": {
        "x": {
          "field": "meio",
          "type": "quantitative"
        },
        "text": {
          "field": "pct",
          "format": ".0%"
        }
      }
    }
  ]
}
```

## 7. Tabelas centralizadas (como está configurado)

Formatar visual › Visual:
1. **Predefinições de estilo** › Nenhum.
2. **Cabeçalhos das colunas** › Alinhamento **Centro** · Fundo `#F1F5F9` · Texto `#2C3E50` · Segoe UI Semibold 10 · Moldar texto ligado.
3. **Elementos de célula / Valores** › Texto `#64748B` · Fundo `#FFFFFF` · Fundo alternado `#F8FAFC` · tamanho 10.
4. **Colunas específicas** › para **cada** coluna: Alinhamento **Centro**, *Aplicar ao cabeçalho* e *Aplicar aos valores* ligados.
5. **Grelha** › Linhas horizontais `#EEF2F6`, verticais desligadas, preenchimento de linha 5.
6. **Formatação condicional (fx, Valor do campo):**
   - Detalhe › *Status* › Cor do texto = `[Cor Status]` · *Espera (min)* › Cor do texto = `[Cor Espera]`.
   - Resumo › *% Cancel.* › Cor de fundo = `[Cor Fundo Cancelamento]` e Cor do texto = `[Cor Fonte Cancelamento]`.
7. Largura das colunas fixada (dimensionamento automático desligado) para a tabela preencher o cartão.

## 8. Passo a passo no Power BI Desktop

1. Instale (uma vez) os visuais da AppSource: **Deneb** e **HTML Content** — o projeto já os referencia; se pedir, clique em *Inserir › Mais visuais › Da AppSource*.
2. Abra `E:\Projetos BI\11 - Vita Care\VittaCare.pbip`.
3. Se aparecer "tabelas incompletas", clique **Atualizar agora** (lê `VittaCare_Database.xlsx`).
4. Confirme em *Exibição de modelo* as 5 relações 1:* (direção única) e `Dim_Calendario` marcada como tabela de datas.
5. Navegue entre abas pelos botões do cabeçalho (Ctrl+clique no modo de edição).
6. Para editar um gráfico Deneb: selecione o visual › `…` › **Editar** — o JSON acima está no separador *Specification*.
7. Para ajustar metas: edite `[Meta Cancelamento]` (10%), `[Meta No-Show]` (5%) e `[Meta Tempo Espera]` (30 min) — cartões, tabelas e scatter atualizam sozinhos.
8. Fonte **Inter**: se estiver instalada no Windows, os cartões/Deneb usam-na; caso contrário caem para Segoe UI/Arial (como indica o brandbook).
9. Guardar: Ctrl+S mantém o formato PBIP (ideal para Git). Para .pbix: *Ficheiro › Guardar como › .pbix*.

## 9. Tooltips HTML animados (report page tooltips)

São 6 páginas ocultas do tipo **Dica de ferramenta** (TT Mês, TT Unidade, TT Especialidade, TT Médico, TT Atendimento e TT Plano). Cada uma tem um único visual **HTML Content** com uma medida da pasta `08 HTML Tooltips`. As páginas ficam escondidas no modo de leitura.

Animações: barra esmeralda que se desenha no topo, brilho radial a pulsar, cabeçalho e números a entrar com fade-up, células da grelha em cascata (stagger), chip com efeito *pop* e barras de meta a crescer.

| Visual | Tooltip | O que mostra |
|---|---|---|
| Evolução do Faturamento (Deneb) | TT Mês | Ranking do mês, variação vs mês anterior e vs ano anterior, consultas, ticket, % de cancelamento, nota, barra contra o melhor mês e especialidade líder |
| Faturamento por Unidade (Deneb) | TT Unidade | % da rede, cidade/UF, variação vs ano anterior, consultas, ticket, nota, espera, pacientes, agenda (realizada/cancelada/no-show), cancelamento vs rede e top médico |
| Donut e Receita por Especialidade (Deneb) | TT Especialidade | % das consultas, ticket vs preço de tabela, nota vs rede, espera, cancelamento e médico destaque |
| Ranking de Médicos (Deneb) | TT Médico | Posição no ranking, estrelas, consultas, faturamento, espera, % no SLA, % de notas 5, no-show e nota vs rede |
| Scatter e Tabela de detalhe | TT Atendimento | ID e data, status, valor, paciente/plano, médico, especialidade, unidade, nota, espera vs meta, média de espera do médico e visitas do paciente |
| Status por Plano (Deneb) e Resumo por Plano (tabela) | TT Plano | % da receita, agendamentos, pacientes, ticket, nota, receita perdida, agenda, segmento em foco e cancelamento vs rede |

**Como funciona:**
- Em cada visual: *Formatar › Geral › Dicas de ferramenta › Tipo = Página do relatório › Página = TT …*.
- Nos gráficos Deneb, a marca usa `"tooltip": true` para manter a identidade da linha. Na Evolução existe uma faixa transparente por mês, porque o hover *nearest/voronoi* perde essa identidade.
- Para editar o visual de uma página TT, mostre as páginas ocultas no modo de edição.

## 10. Painéis de hover nos cartões KPI

Os 12 cartões têm um painel escuro que sobe ao passar o rato (indicador "ⓘ detalhes"), com 4 métricas extra e um insight. As métricas de apoio estão na pasta `07 Apoio Tooltips` (por exemplo: Receita Perdida Estimada, Ticket Convênio, Valor Tabela Médio, Nota com espera ≤30 / >30 min, Top Unidade/Especialidade/Médico/Plano e Melhor Mês).

### HTML TT Mes (TT Mês, 330×352)
```dax
HTML TT Mes =
VAR _mes = SELECTEDVALUE ( Dim_Calendario[Mês Ano], "Período" )
VAR _fat = [Faturamento Total]
VAR _ant = CALCULATE ( [Faturamento Total], DATEADD ( Dim_Calendario[Data], -1, MONTH ) )
VAR _aa = CALCULATE ( [Faturamento Total], SAMEPERIODLASTYEAR ( Dim_Calendario[Data] ) )
VAR _mom = IF ( NOT ISBLANK ( _ant ), DIVIDE ( _fat - _ant, _ant ) )
VAR _yoy = IF ( NOT ISBLANK ( _aa ), DIVIDE ( _fat - _aa, _aa ) )
VAR _media = CALCULATE ( [Faturamento Média Mensal], REMOVEFILTERS ( Dim_Calendario[AnoMes], Dim_Calendario[Mês Ano], Dim_Calendario[Mês], Dim_Calendario[Mês Num] ) )
VAR _dm = DIVIDE ( _fat - _media, _media )
VAR _meses = CALCULATETABLE ( VALUES ( Dim_Calendario[AnoMes] ), REMOVEFILTERS ( Dim_Calendario[AnoMes], Dim_Calendario[Mês Ano], Dim_Calendario[Mês], Dim_Calendario[Mês Num] ) )
VAR _rank = COUNTROWS ( FILTER ( _meses, CALCULATE ( [Faturamento Total], REMOVEFILTERS ( Dim_Calendario[Mês Ano] ) ) > _fat ) ) + 1
VAR _nm = COUNTROWS ( _meses )
VAR _topEsp = [Top Especialidade Faturamento]
VAR _ok = _dm >= 0
RETURN
    "<style>*{box-sizing:border-box;margin:0;padding:0}html,body{background:#22313F;overflow:hidden}.tt{font-family:Inter,'Segoe UI',Arial,sans-serif;width:100vw;height:100vh;padding:14px 16px 12px;color:#fff;background:linear-gradient(155deg,#2C3E50 0%,#22313F 58%,#1B3B3D 100%);position:relative;overflow:hidden;display:flex;flex-direction:column;gap:8px}.tt:before{content:'';position:absolute;right:-60px;top:-60px;width:180px;height:180px;border-radius:50%;background:radial-gradient(circle,rgba(0,203,169,0.32),rgba(0,203,169,0) 70%);animation:glow 3s ease-in-out infinite}.tt:after{content:'';position:absolute;left:0;top:0;height:3px;width:100%;background:linear-gradient(90deg,#00CBA9,#33D6BB 60%,rgba(51,214,187,0));animation:line .9s ease-out both}@keyframes glow{0%,100%{opacity:.55;transform:scale(1)}50%{opacity:1;transform:scale(1.15)}}@keyframes line{from{width:0}}@keyframes up{from{opacity:0;transform:translateY(8px)}to{opacity:1;transform:none}}@keyframes grow{from{width:0}}@keyframes pop{0%{transform:scale(.6);opacity:0}70%{transform:scale(1.08)}100%{transform:scale(1);opacity:1}}.hd{display:flex;justify-content:space-between;align-items:flex-start;gap:8px;position:relative;z-index:1;animation:up .35s ease both}.ey{font-size:9.5px;font-weight:700;letter-spacing:1px;text-transform:uppercase;color:#33D6BB}.ti{font-size:15px;font-weight:700;margin-top:2px;max-width:215px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.chip{font-size:10px;font-weight:700;padding:3px 9px;border-radius:20px;white-space:nowrap;animation:pop .5s .15s ease both}.kp{display:flex;align-items:baseline;gap:8px;position:relative;z-index:1;animation:up .4s .05s ease both}.bv{font-size:24px;font-weight:800;letter-spacing:-.5px}.bs{font-size:10.5px;color:#94A3B8}.gr{display:grid;grid-template-columns:1fr 1fr;gap:6px 8px;position:relative;z-index:1}.gr div{background:rgba(255,255,255,0.06);border:1px solid rgba(255,255,255,0.08);border-radius:9px;padding:5px 9px;animation:up .4s ease both}.gr div:nth-child(1){animation-delay:.08s}.gr div:nth-child(2){animation-delay:.12s}.gr div:nth-child(3){animation-delay:.16s}.gr div:nth-child(4){animation-delay:.2s}.gr div:nth-child(5){animation-delay:.24s}.gr div:nth-child(6){animation-delay:.28s}.gr span{display:block;font-size:8.5px;color:#94A3B8;text-transform:uppercase;letter-spacing:.5px}.gr b{display:block;font-size:12.5px;font-weight:700;margin-top:1px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.br{position:relative;z-index:1;animation:up .4s .3s ease both}.bl{display:flex;justify-content:space-between;font-size:9.5px;color:#CBD5E1;margin-bottom:4px}.bt{height:7px;border-radius:7px;background:rgba(255,255,255,0.10);display:flex;gap:2px;overflow:hidden;position:relative}.bt i{display:block;height:100%;border-radius:7px;animation:grow 1s .35s cubic-bezier(.2,.8,.2,1) both}.bt b{position:absolute;top:0;width:2px;height:100%;background:#fff;opacity:.85}.lg{display:flex;gap:10px;font-size:9px;color:#CBD5E1;margin-top:4px}.lg i{display:inline-block;width:7px;height:7px;border-radius:50%;margin-right:4px}.in{margin-top:auto;display:flex;gap:7px;align-items:center;font-size:10.5px;line-height:1.35;color:#E2E8F0;border-radius:9px;padding:6px 9px;position:relative;z-index:1;animation:up .4s .36s ease both}.in svg{width:14px;height:14px;flex-shrink:0;fill:none;stroke-width:2;stroke-linecap:round;stroke-linejoin:round}.in b{color:#fff}.ok{background:rgba(0,203,169,0.13);border:1px solid rgba(0,203,169,0.30)}.ok svg{stroke:#33D6BB}.al{background:rgba(232,80,91,0.14);border:1px solid rgba(232,80,91,0.35)}.al svg{stroke:#FF8A93}.st{color:#F5B700;letter-spacing:1px;font-size:13px;line-height:1}.pos{color:#33D6BB}.neg{color:#FF8A93}</style>"
        & "<div class='tt'><div class='hd'><div><div class='ey'>Faturamento do mês</div><div class='ti'>" & _mes & "</div></div>" & "<span class='chip' style='" & IF ( _rank <= 3, "background:rgba(0,203,169,0.18);color:#33D6BB", "background:rgba(232,80,91,0.18);color:#FF8A93" ) & "'>" & "#" & _rank & " de " & _nm & "</span>" & "</div>"
        & "<div class='kp'><div class='bv'>" & IF ( ISBLANK ( _fat ), "—", "R$ " & IF ( _fat >= 1000000, FORMAT ( _fat / 1000000, "#,0.0", "pt-BR" ) & " mi", IF ( _fat >= 1000, FORMAT ( _fat / 1000, "#,0.0", "pt-BR" ) & " mil", FORMAT ( _fat, "#,0", "pt-BR" ) ) ) ) & "</div><div class='bs'>" & "vs média " & FORMAT ( _dm, "+0%;-0%", "pt-BR" ) & "</div></div>"
        & "<div class='gr'>"
        & "<div><span>vs mês anterior</span><b>" & ( IF ( ISBLANK ( _mom ), "<span>—</span>", IF ( _mom >= 0, "<span class='pos'>▲ +", "<span class='neg'>▼ -" ) & FORMAT ( ABS ( _mom ), "0.0%", "pt-BR" ) & "</span>" ) ) & "</b></div>"
        & "<div><span>vs ano anterior</span><b>" & ( IF ( ISBLANK ( _yoy ), "<span>—</span>", IF ( _yoy >= 0, "<span class='pos'>▲ +", "<span class='neg'>▼ -" ) & FORMAT ( ABS ( _yoy ), "0.0%", "pt-BR" ) & "</span>" ) ) & "</b></div>"
        & "<div><span>Consultas</span><b>" & ( IF ( ISBLANK ( [Consultas Realizadas] ), "—", FORMAT ( [Consultas Realizadas], "#,0", "pt-BR" ) ) ) & "</b></div>"
        & "<div><span>Ticket médio</span><b>" & ( "R$ " & FORMAT ( [Ticket Médio], "#,0", "pt-BR" ) ) & "</b></div>"
        & "<div><span>% Cancelamento</span><b>" & ( IF ( ISBLANK ( [% Cancelamento] ), "—", FORMAT ( [% Cancelamento], "0.0%", "pt-BR" ) ) ) & "</b></div>"
        & "<div><span>Nota média</span><b>" & ( IF ( ISBLANK ( [Nota Média] ), "—", FORMAT ( [Nota Média], "0.00", "pt-BR" ) ) ) & "</b></div>"
        & "</div>"
        & "<div class='br'><div class='bl'><span>Faturamento vs melhor mês</span><span>" & FORMAT ( DIVIDE ( _fat, MAXX ( _meses, CALCULATE ( [Faturamento Total], REMOVEFILTERS ( Dim_Calendario[Mês Ano] ) ) ) ), "0%", "pt-BR" ) & "</span></div><div class='bt'><i style='width:" & FORMAT ( MAX ( MIN ( DIVIDE ( _fat, MAXX ( _meses, CALCULATE ( [Faturamento Total], REMOVEFILTERS ( Dim_Calendario[Mês Ano] ) ) ) ), 1 ), 0 ) * 100, "0.0", "en-US" ) & "%;background:" & IF ( _dm >= 0, "#00CBA9", "#F28B95" ) & "'></i><b style='left:" & FORMAT ( MAX ( MIN ( DIVIDE ( _media, MAXX ( _meses, CALCULATE ( [Faturamento Total], REMOVEFILTERS ( Dim_Calendario[Mês Ano] ) ) ) ), 1 ), 0 ) * 100, "0.0", "en-US" ) & "%'></b></div></div>"
        & "<div class='in " & IF ( _ok, "ok'><svg viewBox='0 0 24 24'><path d='M22 11.08V12a10 10 0 1 1-5.93-9.14'/><polyline points='22 4 12 14.01 9 11.01'/></svg>", "al'><svg viewBox='0 0 24 24'><path d='M10.29 3.86L1.82 18a2 2 0 0 0 1.71 3h16.94a2 2 0 0 0 1.71-3L13.71 3.86a2 2 0 0 0-3.42 0z'/><line x1='12' y1='9' x2='12' y2='13'/><line x1='12' y1='17' x2='12.01' y2='17'/></svg>" ) & "<span>" & IF ( _dm >= 0, "Mês <b>acima da média</b> do período. ", "Mês <b>abaixo da média</b> do período. " ) & "Especialidade líder: <b>" & _topEsp & "</b>" & "</span></div></div>"
```

### HTML TT Unidade (TT Unidade, 330×352)
```dax
HTML TT Unidade =
VAR _u = SELECTEDVALUE ( Dim_Clinicas[Unidade], "Unidade" )
VAR _local = SELECTEDVALUE ( Dim_Clinicas[Cidade] ) & " · " & SELECTEDVALUE ( Dim_Clinicas[UF] )
VAR _fat = [Faturamento Total]
VAR _rede = CALCULATE ( [Faturamento Total], REMOVEFILTERS ( Dim_Clinicas ) )
VAR _share = DIVIDE ( _fat, _rede )
VAR _canc = [% Cancelamento]
VAR _cancRede = CALCULATE ( [% Cancelamento], REMOVEFILTERS ( Dim_Clinicas ) )
VAR _espera = [Tempo Espera Médio]
VAR _topMed = [Top Médico Faturamento]
VAR _ok = _canc <= _cancRede
RETURN
    "<style>*{box-sizing:border-box;margin:0;padding:0}html,body{background:#22313F;overflow:hidden}.tt{font-family:Inter,'Segoe UI',Arial,sans-serif;width:100vw;height:100vh;padding:14px 16px 12px;color:#fff;background:linear-gradient(155deg,#2C3E50 0%,#22313F 58%,#1B3B3D 100%);position:relative;overflow:hidden;display:flex;flex-direction:column;gap:8px}.tt:before{content:'';position:absolute;right:-60px;top:-60px;width:180px;height:180px;border-radius:50%;background:radial-gradient(circle,rgba(0,203,169,0.32),rgba(0,203,169,0) 70%);animation:glow 3s ease-in-out infinite}.tt:after{content:'';position:absolute;left:0;top:0;height:3px;width:100%;background:linear-gradient(90deg,#00CBA9,#33D6BB 60%,rgba(51,214,187,0));animation:line .9s ease-out both}@keyframes glow{0%,100%{opacity:.55;transform:scale(1)}50%{opacity:1;transform:scale(1.15)}}@keyframes line{from{width:0}}@keyframes up{from{opacity:0;transform:translateY(8px)}to{opacity:1;transform:none}}@keyframes grow{from{width:0}}@keyframes pop{0%{transform:scale(.6);opacity:0}70%{transform:scale(1.08)}100%{transform:scale(1);opacity:1}}.hd{display:flex;justify-content:space-between;align-items:flex-start;gap:8px;position:relative;z-index:1;animation:up .35s ease both}.ey{font-size:9.5px;font-weight:700;letter-spacing:1px;text-transform:uppercase;color:#33D6BB}.ti{font-size:15px;font-weight:700;margin-top:2px;max-width:215px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.chip{font-size:10px;font-weight:700;padding:3px 9px;border-radius:20px;white-space:nowrap;animation:pop .5s .15s ease both}.kp{display:flex;align-items:baseline;gap:8px;position:relative;z-index:1;animation:up .4s .05s ease both}.bv{font-size:24px;font-weight:800;letter-spacing:-.5px}.bs{font-size:10.5px;color:#94A3B8}.gr{display:grid;grid-template-columns:1fr 1fr;gap:6px 8px;position:relative;z-index:1}.gr div{background:rgba(255,255,255,0.06);border:1px solid rgba(255,255,255,0.08);border-radius:9px;padding:5px 9px;animation:up .4s ease both}.gr div:nth-child(1){animation-delay:.08s}.gr div:nth-child(2){animation-delay:.12s}.gr div:nth-child(3){animation-delay:.16s}.gr div:nth-child(4){animation-delay:.2s}.gr div:nth-child(5){animation-delay:.24s}.gr div:nth-child(6){animation-delay:.28s}.gr span{display:block;font-size:8.5px;color:#94A3B8;text-transform:uppercase;letter-spacing:.5px}.gr b{display:block;font-size:12.5px;font-weight:700;margin-top:1px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.br{position:relative;z-index:1;animation:up .4s .3s ease both}.bl{display:flex;justify-content:space-between;font-size:9.5px;color:#CBD5E1;margin-bottom:4px}.bt{height:7px;border-radius:7px;background:rgba(255,255,255,0.10);display:flex;gap:2px;overflow:hidden;position:relative}.bt i{display:block;height:100%;border-radius:7px;animation:grow 1s .35s cubic-bezier(.2,.8,.2,1) both}.bt b{position:absolute;top:0;width:2px;height:100%;background:#fff;opacity:.85}.lg{display:flex;gap:10px;font-size:9px;color:#CBD5E1;margin-top:4px}.lg i{display:inline-block;width:7px;height:7px;border-radius:50%;margin-right:4px}.in{margin-top:auto;display:flex;gap:7px;align-items:center;font-size:10.5px;line-height:1.35;color:#E2E8F0;border-radius:9px;padding:6px 9px;position:relative;z-index:1;animation:up .4s .36s ease both}.in svg{width:14px;height:14px;flex-shrink:0;fill:none;stroke-width:2;stroke-linecap:round;stroke-linejoin:round}.in b{color:#fff}.ok{background:rgba(0,203,169,0.13);border:1px solid rgba(0,203,169,0.30)}.ok svg{stroke:#33D6BB}.al{background:rgba(232,80,91,0.14);border:1px solid rgba(232,80,91,0.35)}.al svg{stroke:#FF8A93}.st{color:#F5B700;letter-spacing:1px;font-size:13px;line-height:1}.pos{color:#33D6BB}.neg{color:#FF8A93}</style>"
        & "<div class='tt'><div class='hd'><div><div class='ey'>Unidade VittaCare</div><div class='ti'>" & _u & "</div></div>" & "<span class='chip' style='" & IF ( _share >= 0.25, "background:rgba(0,203,169,0.18);color:#33D6BB", "background:rgba(232,80,91,0.18);color:#FF8A93" ) & "'>" & FORMAT ( _share, "0.0%", "pt-BR" ) & " da rede" & "</span>" & "</div>"
        & "<div class='kp'><div class='bv'>" & IF ( ISBLANK ( _fat ), "—", "R$ " & IF ( _fat >= 1000000, FORMAT ( _fat / 1000000, "#,0.0", "pt-BR" ) & " mi", IF ( _fat >= 1000, FORMAT ( _fat / 1000, "#,0.0", "pt-BR" ) & " mil", FORMAT ( _fat, "#,0", "pt-BR" ) ) ) ) & "</div><div class='bs'>" & _local & "</div></div>"
        & "<div class='gr'>"
        & "<div><span>Var. ano anterior</span><b>" & ( IF ( ISBLANK ( [Faturamento Var % AA] ), "<span>—</span>", IF ( [Faturamento Var % AA] >= 0, "<span class='pos'>▲ +", "<span class='neg'>▼ -" ) & FORMAT ( ABS ( [Faturamento Var % AA] ), "0.0%", "pt-BR" ) & "</span>" ) ) & "</b></div>"
        & "<div><span>Consultas</span><b>" & ( IF ( ISBLANK ( [Consultas Realizadas] ), "—", FORMAT ( [Consultas Realizadas], "#,0", "pt-BR" ) ) ) & "</b></div>"
        & "<div><span>Ticket médio</span><b>" & ( "R$ " & FORMAT ( [Ticket Médio], "#,0", "pt-BR" ) ) & "</b></div>"
        & "<div><span>Nota média</span><b>" & ( IF ( ISBLANK ( [Nota Média] ), "—", FORMAT ( [Nota Média], "0.00", "pt-BR" ) ) ) & "</b></div>"
        & "<div><span>Espera média</span><b>" & ( IF ( ISBLANK ( _espera ), "—", FORMAT ( _espera, "0.0", "pt-BR" ) ) & " min" ) & "</b></div>"
        & "<div><span>Pacientes</span><b>" & ( IF ( ISBLANK ( [Pacientes Únicos] ), "—", FORMAT ( [Pacientes Únicos], "#,0", "pt-BR" ) ) ) & "</b></div>"
        & "</div>"
        & "<div class='br'><div class='bl'><span>Situação da agenda</span><span>" & FORMAT ( [% Realização], "0%", "pt-BR" ) & " realizadas</span></div><div class='bt'><i style='width:" & FORMAT ( MAX ( MIN ( [% Realização], 1 ), 0 ) * 100, "0.0", "en-US" ) & "%;background:#00CBA9'></i><i style='width:" & FORMAT ( MAX ( MIN ( [% Cancelamento], 1 ), 0 ) * 100, "0.0", "en-US" ) & "%;background:#F28B95'></i><i style='width:" & FORMAT ( MAX ( MIN ( [% No-Show], 1 ), 0 ) * 100, "0.0", "en-US" ) & "%;background:#94A3B8'></i></div><div class='lg'><span><i style='background:#00CBA9'></i>Realizada</span><span><i style='background:#F28B95'></i>Cancelada " & FORMAT ( [% Cancelamento], "0%", "pt-BR" ) & "</span><span><i style='background:#94A3B8'></i>No-show " & FORMAT ( [% No-Show], "0%", "pt-BR" ) & "</span></div></div>"
        & "<div class='in " & IF ( _ok, "ok'><svg viewBox='0 0 24 24'><path d='M22 11.08V12a10 10 0 1 1-5.93-9.14'/><polyline points='22 4 12 14.01 9 11.01'/></svg>", "al'><svg viewBox='0 0 24 24'><path d='M10.29 3.86L1.82 18a2 2 0 0 0 1.71 3h16.94a2 2 0 0 0 1.71-3L13.71 3.86a2 2 0 0 0-3.42 0z'/><line x1='12' y1='9' x2='12' y2='13'/><line x1='12' y1='17' x2='12.01' y2='17'/></svg>" ) & "<span>" & IF ( _canc <= _cancRede, "Cancelamento <b>abaixo</b> da média da rede (", "Cancelamento <b>acima</b> da média da rede (" ) & FORMAT ( _cancRede, "0.0%", "pt-BR" ) & "). Top médico: <b>" & _topMed & "</b>" & "</span></div></div>"
```

### HTML TT Especialidade (TT Especialidade, 330×352)
```dax
HTML TT Especialidade =
VAR _e = SELECTEDVALUE ( Dim_Especialidades[Especialidade], "Especialidade" )
VAR _c = [Consultas Realizadas]
VAR _share = DIVIDE ( _c, CALCULATE ( [Consultas Realizadas], REMOVEFILTERS ( Dim_Especialidades ) ) )
VAR _tab = [Valor Tabela Médio]
VAR _real = [% Realização de Preço]
VAR _nota = [Nota Média]
VAR _notaRede = CALCULATE ( [Nota Média], REMOVEFILTERS ( Dim_Especialidades ) )
VAR _topMed = [Top Médico Faturamento]
VAR _ok = _nota >= _notaRede
RETURN
    "<style>*{box-sizing:border-box;margin:0;padding:0}html,body{background:#22313F;overflow:hidden}.tt{font-family:Inter,'Segoe UI',Arial,sans-serif;width:100vw;height:100vh;padding:14px 16px 12px;color:#fff;background:linear-gradient(155deg,#2C3E50 0%,#22313F 58%,#1B3B3D 100%);position:relative;overflow:hidden;display:flex;flex-direction:column;gap:8px}.tt:before{content:'';position:absolute;right:-60px;top:-60px;width:180px;height:180px;border-radius:50%;background:radial-gradient(circle,rgba(0,203,169,0.32),rgba(0,203,169,0) 70%);animation:glow 3s ease-in-out infinite}.tt:after{content:'';position:absolute;left:0;top:0;height:3px;width:100%;background:linear-gradient(90deg,#00CBA9,#33D6BB 60%,rgba(51,214,187,0));animation:line .9s ease-out both}@keyframes glow{0%,100%{opacity:.55;transform:scale(1)}50%{opacity:1;transform:scale(1.15)}}@keyframes line{from{width:0}}@keyframes up{from{opacity:0;transform:translateY(8px)}to{opacity:1;transform:none}}@keyframes grow{from{width:0}}@keyframes pop{0%{transform:scale(.6);opacity:0}70%{transform:scale(1.08)}100%{transform:scale(1);opacity:1}}.hd{display:flex;justify-content:space-between;align-items:flex-start;gap:8px;position:relative;z-index:1;animation:up .35s ease both}.ey{font-size:9.5px;font-weight:700;letter-spacing:1px;text-transform:uppercase;color:#33D6BB}.ti{font-size:15px;font-weight:700;margin-top:2px;max-width:215px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.chip{font-size:10px;font-weight:700;padding:3px 9px;border-radius:20px;white-space:nowrap;animation:pop .5s .15s ease both}.kp{display:flex;align-items:baseline;gap:8px;position:relative;z-index:1;animation:up .4s .05s ease both}.bv{font-size:24px;font-weight:800;letter-spacing:-.5px}.bs{font-size:10.5px;color:#94A3B8}.gr{display:grid;grid-template-columns:1fr 1fr;gap:6px 8px;position:relative;z-index:1}.gr div{background:rgba(255,255,255,0.06);border:1px solid rgba(255,255,255,0.08);border-radius:9px;padding:5px 9px;animation:up .4s ease both}.gr div:nth-child(1){animation-delay:.08s}.gr div:nth-child(2){animation-delay:.12s}.gr div:nth-child(3){animation-delay:.16s}.gr div:nth-child(4){animation-delay:.2s}.gr div:nth-child(5){animation-delay:.24s}.gr div:nth-child(6){animation-delay:.28s}.gr span{display:block;font-size:8.5px;color:#94A3B8;text-transform:uppercase;letter-spacing:.5px}.gr b{display:block;font-size:12.5px;font-weight:700;margin-top:1px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.br{position:relative;z-index:1;animation:up .4s .3s ease both}.bl{display:flex;justify-content:space-between;font-size:9.5px;color:#CBD5E1;margin-bottom:4px}.bt{height:7px;border-radius:7px;background:rgba(255,255,255,0.10);display:flex;gap:2px;overflow:hidden;position:relative}.bt i{display:block;height:100%;border-radius:7px;animation:grow 1s .35s cubic-bezier(.2,.8,.2,1) both}.bt b{position:absolute;top:0;width:2px;height:100%;background:#fff;opacity:.85}.lg{display:flex;gap:10px;font-size:9px;color:#CBD5E1;margin-top:4px}.lg i{display:inline-block;width:7px;height:7px;border-radius:50%;margin-right:4px}.in{margin-top:auto;display:flex;gap:7px;align-items:center;font-size:10.5px;line-height:1.35;color:#E2E8F0;border-radius:9px;padding:6px 9px;position:relative;z-index:1;animation:up .4s .36s ease both}.in svg{width:14px;height:14px;flex-shrink:0;fill:none;stroke-width:2;stroke-linecap:round;stroke-linejoin:round}.in b{color:#fff}.ok{background:rgba(0,203,169,0.13);border:1px solid rgba(0,203,169,0.30)}.ok svg{stroke:#33D6BB}.al{background:rgba(232,80,91,0.14);border:1px solid rgba(232,80,91,0.35)}.al svg{stroke:#FF8A93}.st{color:#F5B700;letter-spacing:1px;font-size:13px;line-height:1}.pos{color:#33D6BB}.neg{color:#FF8A93}</style>"
        & "<div class='tt'><div class='hd'><div><div class='ey'>Especialidade</div><div class='ti'>" & _e & "</div></div>" & "<span class='chip' style='" & IF ( _share >= 0.18, "background:rgba(0,203,169,0.18);color:#33D6BB", "background:rgba(232,80,91,0.18);color:#FF8A93" ) & "'>" & FORMAT ( _share, "0%", "pt-BR" ) & " das consultas" & "</span>" & "</div>"
        & "<div class='kp'><div class='bv'>" & IF ( ISBLANK ( [Faturamento Total] ), "—", "R$ " & IF ( [Faturamento Total] >= 1000000, FORMAT ( [Faturamento Total] / 1000000, "#,0.0", "pt-BR" ) & " mi", IF ( [Faturamento Total] >= 1000, FORMAT ( [Faturamento Total] / 1000, "#,0.0", "pt-BR" ) & " mil", FORMAT ( [Faturamento Total], "#,0", "pt-BR" ) ) ) ) & "</div><div class='bs'>" & FORMAT ( _c, "#,0", "pt-BR" ) & " consultas realizadas" & "</div></div>"
        & "<div class='gr'>"
        & "<div><span>Ticket médio</span><b>" & ( "R$ " & FORMAT ( [Ticket Médio], "#,0", "pt-BR" ) ) & "</b></div>"
        & "<div><span>Preço tabela</span><b>" & ( "R$ " & FORMAT ( _tab, "#,0", "pt-BR" ) ) & "</b></div>"
        & "<div><span>Nota média</span><b>" & ( IF ( ISBLANK ( _nota ), "—", FORMAT ( _nota, "0.00", "pt-BR" ) ) ) & "</b></div>"
        & "<div><span>Espera média</span><b>" & ( IF ( ISBLANK ( [Tempo Espera Médio] ), "—", FORMAT ( [Tempo Espera Médio], "0.0", "pt-BR" ) ) & " min" ) & "</b></div>"
        & "<div><span>% Cancelamento</span><b>" & ( IF ( ISBLANK ( [% Cancelamento] ), "—", FORMAT ( [% Cancelamento], "0.0%", "pt-BR" ) ) ) & "</b></div>"
        & "<div><span>Pacientes</span><b>" & ( IF ( ISBLANK ( [Pacientes Únicos] ), "—", FORMAT ( [Pacientes Únicos], "#,0", "pt-BR" ) ) ) & "</b></div>"
        & "</div>"
        & "<div class='br'><div class='bl'><span>Nota da especialidade (escala 1–5)</span><span>" & FORMAT ( _nota, "0.00", "pt-BR" ) & " vs rede " & FORMAT ( _notaRede, "0.00", "pt-BR" ) & "</span></div><div class='bt'><i style='width:" & FORMAT ( MAX ( MIN ( DIVIDE ( _nota - 1, 4 ), 1 ), 0 ) * 100, "0.0", "en-US" ) & "%;background:" & IF ( _nota >= _notaRede, "#00CBA9", "#F28B95" ) & "'></i><b style='left:" & FORMAT ( MAX ( MIN ( DIVIDE ( _notaRede - 1, 4 ), 1 ), 0 ) * 100, "0.0", "en-US" ) & "%'></b></div></div>"
        & "<div class='in " & IF ( _ok, "ok'><svg viewBox='0 0 24 24'><path d='M22 11.08V12a10 10 0 1 1-5.93-9.14'/><polyline points='22 4 12 14.01 9 11.01'/></svg>", "al'><svg viewBox='0 0 24 24'><path d='M10.29 3.86L1.82 18a2 2 0 0 0 1.71 3h16.94a2 2 0 0 0 1.71-3L13.71 3.86a2 2 0 0 0-3.42 0z'/><line x1='12' y1='9' x2='12' y2='13'/><line x1='12' y1='17' x2='12.01' y2='17'/></svg>" ) & "<span>" & "Preço praticado <b>" & FORMAT ( _real, "0%", "pt-BR" ) & "</b> da tabela. Médico destaque: <b>" & _topMed & "</b>" & "</span></div></div>"
```

### HTML TT Medico (TT Médico, 330×352)
```dax
HTML TT Medico =
VAR _med = SELECTEDVALUE ( Dim_Medicos[Médico], "Médico" )
VAR _esp = CALCULATE ( SELECTEDVALUE ( Dim_Especialidades[Especialidade] ), CROSSFILTER ( Fato_Atendimentos[ID_Especialidade], Dim_Especialidades[ID_Especialidade], BOTH ) )
VAR _nota = [Nota Média]
VAR _notaRede = CALCULATE ( [Nota Média], REMOVEFILTERS ( Dim_Medicos ), REMOVEFILTERS ( Dim_Especialidades ) )
VAR _lista = CALCULATETABLE ( VALUES ( Dim_Medicos[Médico] ), REMOVEFILTERS ( Dim_Medicos ), REMOVEFILTERS ( Dim_Especialidades ) )
VAR _rank = COUNTROWS ( FILTER ( _lista, CALCULATE ( [Nota Média], REMOVEFILTERS ( Dim_Especialidades ) ) > _nota ) ) + 1
VAR _n = COUNTROWS ( _lista )
VAR _espera = [Tempo Espera Médio]
VAR _ok = _nota >= _notaRede && _espera <= 30
RETURN
    "<style>*{box-sizing:border-box;margin:0;padding:0}html,body{background:#22313F;overflow:hidden}.tt{font-family:Inter,'Segoe UI',Arial,sans-serif;width:100vw;height:100vh;padding:14px 16px 12px;color:#fff;background:linear-gradient(155deg,#2C3E50 0%,#22313F 58%,#1B3B3D 100%);position:relative;overflow:hidden;display:flex;flex-direction:column;gap:8px}.tt:before{content:'';position:absolute;right:-60px;top:-60px;width:180px;height:180px;border-radius:50%;background:radial-gradient(circle,rgba(0,203,169,0.32),rgba(0,203,169,0) 70%);animation:glow 3s ease-in-out infinite}.tt:after{content:'';position:absolute;left:0;top:0;height:3px;width:100%;background:linear-gradient(90deg,#00CBA9,#33D6BB 60%,rgba(51,214,187,0));animation:line .9s ease-out both}@keyframes glow{0%,100%{opacity:.55;transform:scale(1)}50%{opacity:1;transform:scale(1.15)}}@keyframes line{from{width:0}}@keyframes up{from{opacity:0;transform:translateY(8px)}to{opacity:1;transform:none}}@keyframes grow{from{width:0}}@keyframes pop{0%{transform:scale(.6);opacity:0}70%{transform:scale(1.08)}100%{transform:scale(1);opacity:1}}.hd{display:flex;justify-content:space-between;align-items:flex-start;gap:8px;position:relative;z-index:1;animation:up .35s ease both}.ey{font-size:9.5px;font-weight:700;letter-spacing:1px;text-transform:uppercase;color:#33D6BB}.ti{font-size:15px;font-weight:700;margin-top:2px;max-width:215px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.chip{font-size:10px;font-weight:700;padding:3px 9px;border-radius:20px;white-space:nowrap;animation:pop .5s .15s ease both}.kp{display:flex;align-items:baseline;gap:8px;position:relative;z-index:1;animation:up .4s .05s ease both}.bv{font-size:24px;font-weight:800;letter-spacing:-.5px}.bs{font-size:10.5px;color:#94A3B8}.gr{display:grid;grid-template-columns:1fr 1fr;gap:6px 8px;position:relative;z-index:1}.gr div{background:rgba(255,255,255,0.06);border:1px solid rgba(255,255,255,0.08);border-radius:9px;padding:5px 9px;animation:up .4s ease both}.gr div:nth-child(1){animation-delay:.08s}.gr div:nth-child(2){animation-delay:.12s}.gr div:nth-child(3){animation-delay:.16s}.gr div:nth-child(4){animation-delay:.2s}.gr div:nth-child(5){animation-delay:.24s}.gr div:nth-child(6){animation-delay:.28s}.gr span{display:block;font-size:8.5px;color:#94A3B8;text-transform:uppercase;letter-spacing:.5px}.gr b{display:block;font-size:12.5px;font-weight:700;margin-top:1px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.br{position:relative;z-index:1;animation:up .4s .3s ease both}.bl{display:flex;justify-content:space-between;font-size:9.5px;color:#CBD5E1;margin-bottom:4px}.bt{height:7px;border-radius:7px;background:rgba(255,255,255,0.10);display:flex;gap:2px;overflow:hidden;position:relative}.bt i{display:block;height:100%;border-radius:7px;animation:grow 1s .35s cubic-bezier(.2,.8,.2,1) both}.bt b{position:absolute;top:0;width:2px;height:100%;background:#fff;opacity:.85}.lg{display:flex;gap:10px;font-size:9px;color:#CBD5E1;margin-top:4px}.lg i{display:inline-block;width:7px;height:7px;border-radius:50%;margin-right:4px}.in{margin-top:auto;display:flex;gap:7px;align-items:center;font-size:10.5px;line-height:1.35;color:#E2E8F0;border-radius:9px;padding:6px 9px;position:relative;z-index:1;animation:up .4s .36s ease both}.in svg{width:14px;height:14px;flex-shrink:0;fill:none;stroke-width:2;stroke-linecap:round;stroke-linejoin:round}.in b{color:#fff}.ok{background:rgba(0,203,169,0.13);border:1px solid rgba(0,203,169,0.30)}.ok svg{stroke:#33D6BB}.al{background:rgba(232,80,91,0.14);border:1px solid rgba(232,80,91,0.35)}.al svg{stroke:#FF8A93}.st{color:#F5B700;letter-spacing:1px;font-size:13px;line-height:1}.pos{color:#33D6BB}.neg{color:#FF8A93}</style>"
        & "<div class='tt'><div class='hd'><div><div class='ey'>Corpo clínico</div><div class='ti'>" & _med & "</div></div>" & "<span class='chip' style='" & IF ( _rank <= 3, "background:rgba(0,203,169,0.18);color:#33D6BB", "background:rgba(232,80,91,0.18);color:#FF8A93" ) & "'>" & "#" & _rank & " de " & _n & "</span>" & "</div>"
        & "<div class='kp'><div class='bv'>" & IF ( ISBLANK ( _nota ), "—", FORMAT ( _nota, "0.00", "pt-BR" ) ) & " " & "<span class='st'>" & REPT ( "★", ROUND ( _nota, 0 ) ) & "</span><span class='st' style='color:#475569'>" & REPT ( "★", 5 - ROUND ( _nota, 0 ) ) & "</span>" & "</div><div class='bs'>" & _esp & "</div></div>"
        & "<div class='gr'>"
        & "<div><span>Consultas</span><b>" & ( IF ( ISBLANK ( [Consultas Realizadas] ), "—", FORMAT ( [Consultas Realizadas], "#,0", "pt-BR" ) ) ) & "</b></div>"
        & "<div><span>Faturamento</span><b>" & ( IF ( ISBLANK ( [Faturamento Total] ), "—", "R$ " & IF ( [Faturamento Total] >= 1000000, FORMAT ( [Faturamento Total] / 1000000, "#,0.0", "pt-BR" ) & " mi", IF ( [Faturamento Total] >= 1000, FORMAT ( [Faturamento Total] / 1000, "#,0.0", "pt-BR" ) & " mil", FORMAT ( [Faturamento Total], "#,0", "pt-BR" ) ) ) ) ) & "</b></div>"
        & "<div><span>Espera média</span><b>" & ( IF ( ISBLANK ( _espera ), "—", FORMAT ( _espera, "0.0", "pt-BR" ) ) & " min" ) & "</b></div>"
        & "<div><span>No SLA (≤30)</span><b>" & ( IF ( ISBLANK ( [% Dentro SLA Espera] ), "—", FORMAT ( [% Dentro SLA Espera], "0%", "pt-BR" ) ) ) & "</b></div>"
        & "<div><span>Notas 5 ★</span><b>" & ( IF ( ISBLANK ( [% Notas 5] ), "—", FORMAT ( [% Notas 5], "0%", "pt-BR" ) ) ) & "</b></div>"
        & "<div><span>No-show</span><b>" & ( IF ( ISBLANK ( [% No-Show] ), "—", FORMAT ( [% No-Show], "0.0%", "pt-BR" ) ) ) & "</b></div>"
        & "</div>"
        & "<div class='br'><div class='bl'><span>Nota vs média da rede</span><span>" & FORMAT ( _nota - _notaRede, "+0.00;-0.00", "pt-BR" ) & " pts" & "</span></div><div class='bt'><i style='width:" & FORMAT ( MAX ( MIN ( DIVIDE ( _nota - 1, 4 ), 1 ), 0 ) * 100, "0.0", "en-US" ) & "%;background:" & IF ( _nota >= _notaRede, "#00CBA9", "#F28B95" ) & "'></i><b style='left:" & FORMAT ( MAX ( MIN ( DIVIDE ( _notaRede - 1, 4 ), 1 ), 0 ) * 100, "0.0", "en-US" ) & "%'></b></div></div>"
        & "<div class='in " & IF ( _ok, "ok'><svg viewBox='0 0 24 24'><path d='M22 11.08V12a10 10 0 1 1-5.93-9.14'/><polyline points='22 4 12 14.01 9 11.01'/></svg>", "al'><svg viewBox='0 0 24 24'><path d='M10.29 3.86L1.82 18a2 2 0 0 0 1.71 3h16.94a2 2 0 0 0 1.71-3L13.71 3.86a2 2 0 0 0-3.42 0z'/><line x1='12' y1='9' x2='12' y2='13'/><line x1='12' y1='17' x2='12.01' y2='17'/></svg>" ) & "<span>" & IF ( _espera > 30, "Espera média <b>acima da meta</b> de 30 min — risco para a nota.", "Espera <b>dentro da meta</b> de 30 min." ) & " Rede: <b>" & FORMAT ( _notaRede, "0.00", "pt-BR" ) & "</b>" & "</span></div></div>"
```

### HTML TT Atendimento (TT Atendimento, 350×352)
```dax
HTML TT Atendimento =
VAR _id = SELECTEDVALUE ( Fato_Atendimentos[ID_Atendimento] )
VAR _data = MAX ( Fato_Atendimentos[Data Consulta] )
VAR _pac = MAXX ( Fato_Atendimentos, RELATED ( Dim_Pacientes[Paciente] ) )
VAR _plano = MAXX ( Fato_Atendimentos, RELATED ( Dim_Pacientes[Plano de Saúde] ) )
VAR _med = MAXX ( Fato_Atendimentos, RELATED ( Dim_Medicos[Médico] ) )
VAR _esp = MAXX ( Fato_Atendimentos, RELATED ( Dim_Especialidades[Especialidade] ) )
VAR _uni = MAXX ( Fato_Atendimentos, RELATED ( Dim_Clinicas[Unidade] ) )
VAR _st = MAX ( Fato_Atendimentos[Status Consulta] )
VAR _valor = SUM ( Fato_Atendimentos[Valor Cobrado] )
VAR _espera = MAX ( Fato_Atendimentos[Tempo Espera (min)] )
VAR _nota = MAX ( Fato_Atendimentos[Nota Atendimento] )
VAR _idMed = MAX ( Fato_Atendimentos[ID_Medico] )
VAR _idPac = MAX ( Fato_Atendimentos[ID_Paciente] )
VAR _esperaMed = CALCULATE ( [Tempo Espera Médio], REMOVEFILTERS ( Fato_Atendimentos ), REMOVEFILTERS ( Dim_Medicos ), REMOVEFILTERS ( Dim_Especialidades ), Fato_Atendimentos[ID_Medico] = _idMed )
VAR _visitas = CALCULATE ( COUNTROWS ( Fato_Atendimentos ), REMOVEFILTERS ( Fato_Atendimentos ), REMOVEFILTERS ( Dim_Medicos ), REMOVEFILTERS ( Dim_Especialidades ), REMOVEFILTERS ( Dim_Clinicas ), Fato_Atendimentos[ID_Paciente] = _idPac )
VAR _corSt = SWITCH ( _st, "Realizada", "background:rgba(0,203,169,0.18);color:#33D6BB", "Cancelada", "background:rgba(232,80,91,0.18);color:#FF8A93", "background:rgba(245,166,35,0.18);color:#F5C26B" )
VAR _ok = _st = "Realizada" && _espera <= 30
RETURN
    "<style>*{box-sizing:border-box;margin:0;padding:0}html,body{background:#22313F;overflow:hidden}.tt{font-family:Inter,'Segoe UI',Arial,sans-serif;width:100vw;height:100vh;padding:14px 16px 12px;color:#fff;background:linear-gradient(155deg,#2C3E50 0%,#22313F 58%,#1B3B3D 100%);position:relative;overflow:hidden;display:flex;flex-direction:column;gap:8px}.tt:before{content:'';position:absolute;right:-60px;top:-60px;width:180px;height:180px;border-radius:50%;background:radial-gradient(circle,rgba(0,203,169,0.32),rgba(0,203,169,0) 70%);animation:glow 3s ease-in-out infinite}.tt:after{content:'';position:absolute;left:0;top:0;height:3px;width:100%;background:linear-gradient(90deg,#00CBA9,#33D6BB 60%,rgba(51,214,187,0));animation:line .9s ease-out both}@keyframes glow{0%,100%{opacity:.55;transform:scale(1)}50%{opacity:1;transform:scale(1.15)}}@keyframes line{from{width:0}}@keyframes up{from{opacity:0;transform:translateY(8px)}to{opacity:1;transform:none}}@keyframes grow{from{width:0}}@keyframes pop{0%{transform:scale(.6);opacity:0}70%{transform:scale(1.08)}100%{transform:scale(1);opacity:1}}.hd{display:flex;justify-content:space-between;align-items:flex-start;gap:8px;position:relative;z-index:1;animation:up .35s ease both}.ey{font-size:9.5px;font-weight:700;letter-spacing:1px;text-transform:uppercase;color:#33D6BB}.ti{font-size:15px;font-weight:700;margin-top:2px;max-width:215px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.chip{font-size:10px;font-weight:700;padding:3px 9px;border-radius:20px;white-space:nowrap;animation:pop .5s .15s ease both}.kp{display:flex;align-items:baseline;gap:8px;position:relative;z-index:1;animation:up .4s .05s ease both}.bv{font-size:24px;font-weight:800;letter-spacing:-.5px}.bs{font-size:10.5px;color:#94A3B8}.gr{display:grid;grid-template-columns:1fr 1fr;gap:6px 8px;position:relative;z-index:1}.gr div{background:rgba(255,255,255,0.06);border:1px solid rgba(255,255,255,0.08);border-radius:9px;padding:5px 9px;animation:up .4s ease both}.gr div:nth-child(1){animation-delay:.08s}.gr div:nth-child(2){animation-delay:.12s}.gr div:nth-child(3){animation-delay:.16s}.gr div:nth-child(4){animation-delay:.2s}.gr div:nth-child(5){animation-delay:.24s}.gr div:nth-child(6){animation-delay:.28s}.gr span{display:block;font-size:8.5px;color:#94A3B8;text-transform:uppercase;letter-spacing:.5px}.gr b{display:block;font-size:12.5px;font-weight:700;margin-top:1px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.br{position:relative;z-index:1;animation:up .4s .3s ease both}.bl{display:flex;justify-content:space-between;font-size:9.5px;color:#CBD5E1;margin-bottom:4px}.bt{height:7px;border-radius:7px;background:rgba(255,255,255,0.10);display:flex;gap:2px;overflow:hidden;position:relative}.bt i{display:block;height:100%;border-radius:7px;animation:grow 1s .35s cubic-bezier(.2,.8,.2,1) both}.bt b{position:absolute;top:0;width:2px;height:100%;background:#fff;opacity:.85}.lg{display:flex;gap:10px;font-size:9px;color:#CBD5E1;margin-top:4px}.lg i{display:inline-block;width:7px;height:7px;border-radius:50%;margin-right:4px}.in{margin-top:auto;display:flex;gap:7px;align-items:center;font-size:10.5px;line-height:1.35;color:#E2E8F0;border-radius:9px;padding:6px 9px;position:relative;z-index:1;animation:up .4s .36s ease both}.in svg{width:14px;height:14px;flex-shrink:0;fill:none;stroke-width:2;stroke-linecap:round;stroke-linejoin:round}.in b{color:#fff}.ok{background:rgba(0,203,169,0.13);border:1px solid rgba(0,203,169,0.30)}.ok svg{stroke:#33D6BB}.al{background:rgba(232,80,91,0.14);border:1px solid rgba(232,80,91,0.35)}.al svg{stroke:#FF8A93}.st{color:#F5B700;letter-spacing:1px;font-size:13px;line-height:1}.pos{color:#33D6BB}.neg{color:#FF8A93}</style>"
        & "<div class='tt'><div class='hd'><div><div class='ey'>Atendimento</div><div class='ti'>" & "#" & _id & " · " & FORMAT ( _data, "dd/mm/yyyy", "pt-BR" ) & "</div></div>" & "<span class='chip' style='" & _corSt & "'>" & _st & "</span>" & "</div>"
        & "<div class='kp'><div class='bv'>" & IF ( _st = "Realizada", "R$ " & FORMAT ( _valor, "#,0.00", "pt-BR" ), "R$ 0,00" ) & "</div><div class='bs'>" & _pac & " · " & _plano & "</div></div>"
        & "<div class='gr'>"
        & "<div><span>Médico</span><b>" & ( _med ) & "</b></div>"
        & "<div><span>Especialidade</span><b>" & ( _esp ) & "</b></div>"
        & "<div><span>Unidade</span><b>" & ( _uni ) & "</b></div>"
        & "<div><span>Nota</span><b>" & ( IF ( ISBLANK ( _nota ), "—", "<span class='st'>" & REPT ( "★", ROUND ( _nota, 0 ) ) & "</span><span class='st' style='color:#475569'>" & REPT ( "★", 5 - ROUND ( _nota, 0 ) ) & "</span>" ) ) & "</b></div>"
        & "<div><span>Espera</span><b>" & ( IF ( _st = "Realizada", _espera & " min", "—" ) ) & "</b></div>"
        & "<div><span>Visitas do paciente</span><b>" & ( _visitas & " no total" ) & "</b></div>"
        & "</div>"
        & "<div class='br'><div class='bl'><span>Espera vs meta (30 min) · escala 60 min</span><span>" & IF ( _st = "Realizada", _espera & " min · média do médico " & FORMAT ( _esperaMed, "0", "pt-BR" ) & " min", "consulta não realizada" ) & "</span></div><div class='bt'><i style='width:" & FORMAT ( MAX ( MIN ( IF ( _st = "Realizada", DIVIDE ( _espera, 60 ), 0 ), 1 ), 0 ) * 100, "0.0", "en-US" ) & "%;background:" & IF ( _espera > 30, "#F28B95", "#00CBA9" ) & "'></i><b style='left:" & FORMAT ( MAX ( MIN ( DIVIDE ( 30, 60 ), 1 ), 0 ) * 100, "0.0", "en-US" ) & "%'></b></div></div>"
        & "<div class='in " & IF ( _ok, "ok'><svg viewBox='0 0 24 24'><path d='M22 11.08V12a10 10 0 1 1-5.93-9.14'/><polyline points='22 4 12 14.01 9 11.01'/></svg>", "al'><svg viewBox='0 0 24 24'><path d='M10.29 3.86L1.82 18a2 2 0 0 0 1.71 3h16.94a2 2 0 0 0 1.71-3L13.71 3.86a2 2 0 0 0-3.42 0z'/><line x1='12' y1='9' x2='12' y2='13'/><line x1='12' y1='17' x2='12.01' y2='17'/></svg>" ) & "<span>" & SWITCH ( TRUE (), _st = "Cancelada", "Consulta <b>cancelada</b> — sem receita gerada.", _st = "No-Show", "Paciente <b>faltou</b> sem aviso — avaliar lembrete/confirmação.", _espera > 30, "Espera <b>" & ( _espera - 30 ) & " min acima</b> da meta.", "Atendimento <b>dentro da meta</b> de espera." ) & "</span></div></div>"
```

### HTML TT Plano (TT Plano, 330×318)
```dax
HTML TT Plano =
VAR _p = SELECTEDVALUE ( Dim_Pacientes[Plano de Saúde], "Plano" )
VAR _stSel = SELECTEDVALUE ( Fato_Atendimentos[Status Consulta] )
VAR _ag = CALCULATE ( [Agendamentos], REMOVEFILTERS ( Fato_Atendimentos[Status Consulta] ) )
VAR _r = CALCULATE ( [% Realização], REMOVEFILTERS ( Fato_Atendimentos[Status Consulta] ) )
VAR _c = CALCULATE ( [% Cancelamento], REMOVEFILTERS ( Fato_Atendimentos[Status Consulta] ) )
VAR _n = CALCULATE ( [% No-Show], REMOVEFILTERS ( Fato_Atendimentos[Status Consulta] ) )
VAR _fat = CALCULATE ( [Faturamento Total], REMOVEFILTERS ( Fato_Atendimentos[Status Consulta] ) )
VAR _share = DIVIDE ( _fat, CALCULATE ( [Faturamento Total], REMOVEFILTERS ( Dim_Pacientes ), REMOVEFILTERS ( Fato_Atendimentos[Status Consulta] ) ) )
VAR _perda = CALCULATE ( [Receita Perdida Estimada], REMOVEFILTERS ( Fato_Atendimentos[Status Consulta] ) )
VAR _cancRede = CALCULATE ( [% Cancelamento], REMOVEFILTERS ( Dim_Pacientes ), REMOVEFILTERS ( Fato_Atendimentos[Status Consulta] ) )
VAR _ok = _c <= _cancRede
RETURN
    "<style>*{box-sizing:border-box;margin:0;padding:0}html,body{background:#22313F;overflow:hidden}.tt{font-family:Inter,'Segoe UI',Arial,sans-serif;width:100vw;height:100vh;padding:14px 16px 12px;color:#fff;background:linear-gradient(155deg,#2C3E50 0%,#22313F 58%,#1B3B3D 100%);position:relative;overflow:hidden;display:flex;flex-direction:column;gap:8px}.tt:before{content:'';position:absolute;right:-60px;top:-60px;width:180px;height:180px;border-radius:50%;background:radial-gradient(circle,rgba(0,203,169,0.32),rgba(0,203,169,0) 70%);animation:glow 3s ease-in-out infinite}.tt:after{content:'';position:absolute;left:0;top:0;height:3px;width:100%;background:linear-gradient(90deg,#00CBA9,#33D6BB 60%,rgba(51,214,187,0));animation:line .9s ease-out both}@keyframes glow{0%,100%{opacity:.55;transform:scale(1)}50%{opacity:1;transform:scale(1.15)}}@keyframes line{from{width:0}}@keyframes up{from{opacity:0;transform:translateY(8px)}to{opacity:1;transform:none}}@keyframes grow{from{width:0}}@keyframes pop{0%{transform:scale(.6);opacity:0}70%{transform:scale(1.08)}100%{transform:scale(1);opacity:1}}.hd{display:flex;justify-content:space-between;align-items:flex-start;gap:8px;position:relative;z-index:1;animation:up .35s ease both}.ey{font-size:9.5px;font-weight:700;letter-spacing:1px;text-transform:uppercase;color:#33D6BB}.ti{font-size:15px;font-weight:700;margin-top:2px;max-width:215px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.chip{font-size:10px;font-weight:700;padding:3px 9px;border-radius:20px;white-space:nowrap;animation:pop .5s .15s ease both}.kp{display:flex;align-items:baseline;gap:8px;position:relative;z-index:1;animation:up .4s .05s ease both}.bv{font-size:24px;font-weight:800;letter-spacing:-.5px}.bs{font-size:10.5px;color:#94A3B8}.gr{display:grid;grid-template-columns:1fr 1fr;gap:6px 8px;position:relative;z-index:1}.gr div{background:rgba(255,255,255,0.06);border:1px solid rgba(255,255,255,0.08);border-radius:9px;padding:5px 9px;animation:up .4s ease both}.gr div:nth-child(1){animation-delay:.08s}.gr div:nth-child(2){animation-delay:.12s}.gr div:nth-child(3){animation-delay:.16s}.gr div:nth-child(4){animation-delay:.2s}.gr div:nth-child(5){animation-delay:.24s}.gr div:nth-child(6){animation-delay:.28s}.gr span{display:block;font-size:8.5px;color:#94A3B8;text-transform:uppercase;letter-spacing:.5px}.gr b{display:block;font-size:12.5px;font-weight:700;margin-top:1px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis}.br{position:relative;z-index:1;animation:up .4s .3s ease both}.bl{display:flex;justify-content:space-between;font-size:9.5px;color:#CBD5E1;margin-bottom:4px}.bt{height:7px;border-radius:7px;background:rgba(255,255,255,0.10);display:flex;gap:2px;overflow:hidden;position:relative}.bt i{display:block;height:100%;border-radius:7px;animation:grow 1s .35s cubic-bezier(.2,.8,.2,1) both}.bt b{position:absolute;top:0;width:2px;height:100%;background:#fff;opacity:.85}.lg{display:flex;gap:10px;font-size:9px;color:#CBD5E1;margin-top:4px}.lg i{display:inline-block;width:7px;height:7px;border-radius:50%;margin-right:4px}.in{margin-top:auto;display:flex;gap:7px;align-items:center;font-size:10.5px;line-height:1.35;color:#E2E8F0;border-radius:9px;padding:6px 9px;position:relative;z-index:1;animation:up .4s .36s ease both}.in svg{width:14px;height:14px;flex-shrink:0;fill:none;stroke-width:2;stroke-linecap:round;stroke-linejoin:round}.in b{color:#fff}.ok{background:rgba(0,203,169,0.13);border:1px solid rgba(0,203,169,0.30)}.ok svg{stroke:#33D6BB}.al{background:rgba(232,80,91,0.14);border:1px solid rgba(232,80,91,0.35)}.al svg{stroke:#FF8A93}.st{color:#F5B700;letter-spacing:1px;font-size:13px;line-height:1}.pos{color:#33D6BB}.neg{color:#FF8A93}</style>"
        & "<div class='tt'><div class='hd'><div><div class='ey'>Plano de saúde</div><div class='ti'>" & _p & "</div></div>" & "<span class='chip' style='" & IF ( _share >= 0.2, "background:rgba(0,203,169,0.18);color:#33D6BB", "background:rgba(232,80,91,0.18);color:#FF8A93" ) & "'>" & FORMAT ( _share, "0%", "pt-BR" ) & " da receita" & "</span>" & "</div>"
        & "<div class='kp'><div class='bv'>" & IF ( ISBLANK ( _fat ), "—", "R$ " & IF ( _fat >= 1000000, FORMAT ( _fat / 1000000, "#,0.0", "pt-BR" ) & " mi", IF ( _fat >= 1000, FORMAT ( _fat / 1000, "#,0.0", "pt-BR" ) & " mil", FORMAT ( _fat, "#,0", "pt-BR" ) ) ) ) & "</div><div class='bs'>" & FORMAT ( _ag, "#,0", "pt-BR" ) & " agendamentos" & "</div></div>"
        & "<div class='gr'>"
        & "<div><span>Pacientes</span><b>" & ( FORMAT ( CALCULATE ( [Pacientes Únicos], REMOVEFILTERS ( Fato_Atendimentos[Status Consulta] ) ), "#,0", "pt-BR" ) ) & "</b></div>"
        & "<div><span>Ticket médio</span><b>" & ( "R$ " & FORMAT ( CALCULATE ( [Ticket Médio], REMOVEFILTERS ( Fato_Atendimentos[Status Consulta] ) ), "#,0", "pt-BR" ) ) & "</b></div>"
        & "<div><span>Nota média</span><b>" & ( FORMAT ( CALCULATE ( [Nota Média], REMOVEFILTERS ( Fato_Atendimentos[Status Consulta] ) ), "0.00", "pt-BR" ) ) & "</b></div>"
        & "<div><span>Receita perdida</span><b>" & ( IF ( ISBLANK ( _perda ), "—", "R$ " & IF ( _perda >= 1000000, FORMAT ( _perda / 1000000, "#,0.0", "pt-BR" ) & " mi", IF ( _perda >= 1000, FORMAT ( _perda / 1000, "#,0.0", "pt-BR" ) & " mil", FORMAT ( _perda, "#,0", "pt-BR" ) ) ) ) ) & "</b></div>"
        & "</div>"
        & "<div class='br'><div class='bl'><span>Situação da agenda</span><span>" & FORMAT ( _r, "0%", "pt-BR" ) & " realizadas</span></div><div class='bt'><i style='width:" & FORMAT ( MAX ( MIN ( _r, 1 ), 0 ) * 100, "0.0", "en-US" ) & "%;background:#00CBA9'></i><i style='width:" & FORMAT ( MAX ( MIN ( _c, 1 ), 0 ) * 100, "0.0", "en-US" ) & "%;background:#F28B95'></i><i style='width:" & FORMAT ( MAX ( MIN ( _n, 1 ), 0 ) * 100, "0.0", "en-US" ) & "%;background:#94A3B8'></i></div><div class='lg'><span><i style='background:#00CBA9'></i>Realizada</span><span><i style='background:#F28B95'></i>Cancelada " & FORMAT ( _c, "0%", "pt-BR" ) & "</span><span><i style='background:#94A3B8'></i>No-show " & FORMAT ( _n, "0%", "pt-BR" ) & "</span></div></div>"
        & "<div class='in " & IF ( _ok, "ok'><svg viewBox='0 0 24 24'><path d='M22 11.08V12a10 10 0 1 1-5.93-9.14'/><polyline points='22 4 12 14.01 9 11.01'/></svg>", "al'><svg viewBox='0 0 24 24'><path d='M10.29 3.86L1.82 18a2 2 0 0 0 1.71 3h16.94a2 2 0 0 0 1.71-3L13.71 3.86a2 2 0 0 0-3.42 0z'/><line x1='12' y1='9' x2='12' y2='13'/><line x1='12' y1='17' x2='12.01' y2='17'/></svg>" ) & "<span>" & IF ( ISBLANK ( _stSel ) || HASONEVALUE ( Fato_Atendimentos[Status Consulta] ) = FALSE (), "", "Segmento em foco: <b>" & _stSel & "</b> · " ) & IF ( _c <= _cancRede, "Cancelamento <b>abaixo</b> da rede (", "Cancelamento <b>acima</b> da rede (" ) & FORMAT ( _cancRede, "0.0%", "pt-BR" ) & ")" & "</span></div></div>"
```
