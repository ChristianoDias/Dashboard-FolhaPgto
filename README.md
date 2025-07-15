Ao cumpriment-lo(a) apresento,

Este projeto de Business Intelligence tem como objetivo analisar o absenteísmo na folha de pagamento, identificando padrões, causas e possíveis impactos por departamento se ocorridas por faltas ou atestados, levando o impacto no mês pelo turno(diurno ou noturno). O dashboard foi desenvolvido utilizando SQL Server para estruturação de dados, DAX para criação de medidas, e Power BI para visualização.Inciando pelo link do Dashboard interativo: https://app.powerbi.com/view?r=eyJrIjoiZTY3OGEyMjMtZmE3MC00ZDYzLWFlYjgtNWJiNDFmNWU0ZmJkIiwidCI6ImEzNGRiODZkLTVhYTEtNGVkNi04MTFlLTU2NDhiOWE2ZTY1NiJ9

Sobre as regras para estruturação das tabelas em SQL SERVER: A planilha hospitalar foi estruturada para gestão mensal de colaboradores, assegurando dados consistentes e realistas para o ano de 2024. Cada registro inclui cartão ponto único, carga horária mensal (CH) variando entre 120 e 220 horas, e salário base fixo alinhado a essa carga horária, com valores entre R$ 2.000 e R$ 25.000. Setores e gerências seguem nomenclatura padronizada com prefixos como “Depto.” e “GR.”, mantendo coerência organizacional. A classificação de horário contempla turnos diurno e noturno, sendo restrita ao diurno para áreas como RH, Contabilidade e Financeiro.O controle de absenteísmo considera faltas e atestados em valores inteiros, distribuídos entre 1% e 10% dos colaboradores por setor, com exceção do RH, que não apresenta ausências. Uma coluna específica indica quando a soma de ausências atinge 50% da carga horária, facilitando a sinalização previdenciária. Horas extras são limitadas a 50% da carga horária mensal e aplicadas exclusivamente à cobertura de ausências dentro do mesmo setor. A substituição é vinculada diretamente ao colaborador ausente, com preenchimento dos campos de cartão ponto e horário do substituído, garantindo rastreabilidade e alinhamento operacional. Esta modelagem permite um controle detalhado e confiável das jornadas e substituições, apoiando a gestão de recursos humanos de forma estruturada e padronizada.

Ex de tabelas: Tabelas Folha de Pagamento

Desenvolvimento de queries SQL para seleção, transformação e junção dos dados necessários ao modelo: Um exemplo de querey ao projeto, foi desenvolvida uma consulta em SQL Server para estimar o impacto do absenteísmo, considerando o custo adicional de horas extras necessárias para cobertura. O cálculo aplica um fator de hora extra conforme o turno do colaborador substituído: para o turno diurno, utiliza-se um fator de 0,5 sobre as horas extras do substituto, refletindo cobertura parcial. Para o turno noturno, o fator é 1,0, prevendo cobertura integral devido à menor disponibilidade de pessoal nesse horário. Além disso, o modelo impõe um limite de 50% da carga horária mensal do substituto para as horas extras, garantindo conformidade com restrições trabalhistas e evitando excedentes. A consulta também assegura que a substituição ocorra entre colaboradores do mesmo cargo e departamento, mantendo alinhamento funcional e consistência nos custos. Essa lógica permite calcular de forma mais precisa o custo do absenteísmo e oferece suporte técnico sólido para decisões estratégicas de gestão de equipe e orçamento.

Exemplo da query utilizada:

SELECT
    HE."Cartão Ponto" AS CP_Substituido,
    HE.Mês AS Mes_Ocorrencia,
    U_Substituido.Setor AS Departamento_Origem_Substituido,
    HE."CP DO SUBSTITUIDO" AS CP_Substituto,
    HE.Faltas AS HorasFalta_HE,
    HE.Atestado AS HorasAtestado_HE,
    C_Substituto."Salário Base" AS SalarioBase_Substituto,
    C_Substituto.CH AS CH_Substituto,
    D_Substituto.Turno AS Turno_Substituto,
    (C_Substituto."Salário Base" / NULLIF(C_Substituto.CH, 0)) AS ValorHoraNormal_Substituto,
    CASE
        WHEN D_Substituto.Turno LIKE '%DIURNO%' THEN 0.5
        WHEN D_Substituto.Turno LIKE '%NOTURNO%' THEN 1.0
        ELSE 0.0
    END AS FatorMultiplicacaoTurno,
    (
        (C_Substituto."Salário Base" / NULLIF(C_Substituto.CH, 0)) *
        CASE
            WHEN D_Substituto.Turno LIKE '%DIURNO%' THEN 0.5
            WHEN D_Substituto.Turno LIKE '%NOTURNO%' THEN 1.0
            ELSE 0.0
        END
        +
        (C_Substituto."Salário Base" / NULLIF(C_Substituto.CH, 0))
    ) * (ISNULL(HE.Faltas, 0) + ISNULL(HE.Atestado, 0)) AS Valor_HE_Individual_Calculado
FROM [dbo].[HORAS EXTRAS] AS HE
INNER JOIN [dbo].[Dados] AS D_Substituto
    ON HE."CP DO SUBSTITUIDO" = D_Substituto."Cartão Ponto"
INNER JOIN [dbo].[Cargo] AS C_Substituto
    ON HE."CP DO SUBSTITUIDO" = C_Substituto."Cartão Ponto"
LEFT JOIN [dbo].[Unidade ] AS U_Substituido
    ON HE."Cartão Ponto" = U_Substituido."Cartão Ponto"
WHERE C_Substituto.CH IS NOT NULL AND C_Substituto.CH > 0
), TotalHorasExtrasPorDepartamentoMensal AS ( SELECT VHE.Mes_Ocorrencia AS Mes_Nome, ISNULL(VHE.Departamento_Origem_Substituido, 'Departamento Não Identificado') AS Nome_Departamento, ROUND(SUM(ISNULL(VHE.Valor_HE_Individual_Calculado, 0)), 2) AS Valor_Total_Horas_Extras_Depto FROM CalculoValorHE_porSubstituicao AS VHE GROUP BY VHE.Mes_Ocorrencia, ISNULL(VHE.Departamento_Origem_Substituido, 'Departamento Não Identificado') ) SELECT MP.Mes AS Mes_Nome, AD.Nome_Departamento, ISNULL(THE.Valor_Total_Horas_Extras_Depto, 0.00) AS Valor_Horas_Extras_Departamento, ISNULL(FB.Valor_Folha_Bruta_Departamento, 0.00) AS Valor_Folha_Bruta_Departamento, ROUND( CASE WHEN ISNULL(FB.Valor_Folha_Bruta_Departamento, 0) > 0 THEN (ISNULL(THE.Valor_Total_Horas_Extras_Depto, 0.00) / FB.Valor_Folha_Bruta_Departamento) * 100 ELSE 0.00 END, 2) AS Percentual_Horas_Extras_Sobre_Folha_Bruta FROM MesesDoPeriodo AS MP CROSS JOIN ( SELECT DISTINCT Setor AS Nome_Departamento FROM [dbo].[Unidade ] ) AS AD LEFT JOIN TotalHorasExtrasPorDepartamentoMensal AS THE ON MP.Mes = THE.Mes_Nome AND AD.Nome_Departamento = THE.Nome_Departamento LEFT JOIN [dbo].[vw_FolhaBrutaPorDepartamentoMensal] AS FB ON MP.Mes = FB.Mes_Nome AND AD.Nome_Departamento = FB.Nome_Departamento;

Resultado: Resultado calculo horas extras

Conclusão

A análise realizada demonstra de forma clara o custo bruto da folha por mês e por departamento, evidenciando o peso financeiro do absenteísmo em sua composição percentual. Destaca-se que o departamento de Contabilidade apresentou o maior custo absoluto e a maior proporção relativa de absenteísmo, indicando um impacto financeiro significativo nessa área. Entretanto, o gráfico estático revela que, em vários meses, outros departamentos — como o Administrativo em abril — registraram volumes mais elevados de horas extras. Esse resultado sugere não apenas maior demanda de trabalho pontual, mas também a influência de uma folha salarial proporcionalmente menor, o que requer atenção para o dimensionamento adequado das equipes e o gerenciamento de custos adicionais. Em relação aos fatores que compõem o absenteísmo, observa-se que os atestados médicos prevalecem sobre as horas faltas. Esse comportamento, esperado em ambiente hospitalar ou corporativo, reforça a necessidade de monitoramento detalhado, pois atestados tendem a representar ausências justificadas, porém frequentes, que podem indicar fragilidades em saúde ocupacional, ergonomia ou até mesmo em clima organizacional, exigindo abordagens preventivas mais estruturadas. Por fim, a análise por turno evidencia maior concentração de absenteísmo no período diurno, possivelmente associado às demandas e pressões típicas de horários comerciais. Esse dado indica a necessidade de estratégias específicas para mitigar ausências nesse turno, como adequação de escalas, programas de bem-estar e ações de engajamento voltadas para a realidade do expediente diurno. Em conjunto, esses resultados oferecem subsídios sólidos para o planejamento de ações corretivas e preventivas, visando reduzir custos, melhorar a alocação de recursos humanos e promover maior eficiência operacional.

Agradecimentos: Agradeço pela atenção dedicada a esta análise e coloco-me inteiramente à disposição para eventuais esclarecimentos, discussões adicionais ou apoio na definição e implantação de estratégias que visem à melhoria contínua dos resultados apresentados.
