# **Código de importação de API do Portal de Transparência do Governo Federal**:


*   O intuito deste trabalho é estudar algumas aplicações e integrações atrávez das API's disponíveis do Governo Federal - Instituição: UNIFESP-UNIVERSIDADE FEDERAL DE SAO PAULO (data inicial: 01/01/2022)




!pip install pandas
import requests
import pandas as pd

def obter_dados_api(codigo_orgao, data_inicial):
    url = "https://api.portaldatransparencia.gov.br/api-de-dados/contratos"
    chave_api = "x"

    params = {"codigoOrgao": codigo_orgao, "quantidade": 100, "dataInicial": data_inicial, "pagina": 1}
    headers = {"accept": "*/*", "chave-api-dados": chave_api}

    dados_paginas = []

    while True:
        try:
            response = requests.get(url, params=params, headers=headers)
            response.raise_for_status()
            dados_json = response.json()

            if not dados_json:
                break

            dados_paginas.extend(dados_json)
            params["pagina"] += 1

        except requests.exceptions.RequestException as e:
            print("Erro ao fazer a requisição:", e)
            break

    return dados_paginas

def criar_dataframe():
    codigo_orgao = "26262"
    data_inicial = "01/01/2022"

    dados_contratos = obter_dados_api(codigo_orgao, data_inicial)

    if dados_contratos:
        df = pd.DataFrame(dados_contratos)
        return df
    else:
        return None

if __name__ == "__main__":
    df = criar_dataframe()

df.rename(columns={'id': 'id_inicial', 'objeto': 'objeto_contrato'}, inplace=True)
df.shape

# **Visualização dos dados em formato de tabela**




# Normalizar as colunas que contem dicionario de dados
compras_df = pd.json_normalize(df['compra'])
unidade_gestora_df = pd.json_normalize(df['unidadeGestora'])
fornecedor_df = pd.json_normalize(df['fornecedor'])

# Concatenar o DataFrame original (df) com os DataFrames das colunas de compras e unidadeGestora
df_normalizado = pd.concat([df, compras_df, fornecedor_df,unidade_gestora_df], axis=1)
df_normalizado.drop(['compra', 'unidadeGestora','orgaoVinculado.codigoSIAFI', 'orgaoVinculado.cnpj',
       'orgaoVinculado.sigla', 'orgaoVinculado.nome', 'orgaoMaximo.codigo',
       'orgaoMaximo.sigla', 'orgaoMaximo.nome','numeroProcesso','unidadeGestoraCompras','descricaoPoder','fornecedor','id','objeto','cpfFormatado','numeroInscricaoSocial','razaoSocialReceita','nomeFantasiaReceita','tipo'], axis=1, inplace=True)

# Transpor o DataFrame para identificar colunas duplicadas
df_t = df_normalizado.T

# Identificar e manter apenas as colunas não duplicadas
df_normalizado = df_t.loc[:, ~df_t.columns.duplicated()]
df_final = df_normalizado.T
df_final

# Remover duplicidade 
df_final.rename(columns={'codigo': 'uge','nome':'nome_uge'}, inplace=True)
df_final

# Excluir linhas duplicadas com valores iguais em duas colunas específicas
colunas_para_verificar_duplicatas = ['valorInicialCompra', 'valorFinalCompra']

df_final_corrigido = df_final.drop_duplicates(subset=colunas_para_verificar_duplicatas, keep='first')

df_final_corrigido

# **Filtrar apenas a Descritiva dos Contratos**



# Filtrar apenas uma "UGe"
df_153031= df_final_corrigido[df_final_corrigido['uge'] == '153031']
df_153031

# Criar um novo dataframe contendo a contagem de objetos dos contratos
df_contagem_objeto = df_final_corrigido.groupby('objeto_contrato')['id_inicial'].count().reset_index()

# Renomear colunas do novo dataframe 
df_contagem_objeto.columns = ['objeto_contrato', 'Quantidade de Contratos']

# Ordenando o dataframe pela coluna 'Quantidade de Contratos' em ordem decrescente
df_contagem_objeto = df_contagem_objeto.sort_values(by='Quantidade de Contratos',ascending=False)

df_contagem_objeto

# Criar um novo dataframe contendo a contagem de objetos do contrato por modalidade de compra da UGE 153031.
df_contagem_objeto = df_153031.groupby('objeto_contrato')['id_inicial'].count().reset_index()

# Renomear as colunas do novo dataframe para facilitar a compreensão
df_contagem_objeto.columns = ['objeto_contrato', 'Quantidade de Contratos']

# Ordenandar o dataframe pela coluna 'Quantidade de Contratos' em ordem decrescente
df_contagem_objeto = df_contagem_objeto.sort_values(by='Quantidade de Contratos',ascending=False)

df_contagem_objeto.set_index('objeto_contrato')


# **Criando uma Tabela com as Modalidades de Compra e Quantidade**:



#Unindo os dataframes

df_153031= df_final_corrigido[df_final_corrigido['uge'] == '153031']
df_153031

df_contagem_modalidade = df_final_corrigido.groupby('modalidadeCompra')['id_inicial'].count().reset_index()

df_contagem_modalidade.columns = ['Modalidade de Compra', 'Quantidade de Contratos']

df_contagem_modalidade = df_contagem_modalidade.sort_values(by='Quantidade de Contratos',ascending=False)

df_contagem_modalidade

df_contagem_modalidade = df_153031.groupby('modalidadeCompra')['id_inicial'].count().reset_index()

df_contagem_modalidade.columns = ['Modalidade de Compra', 'Quantidade de Contratos']

df_contagem_modalidade = df_contagem_modalidade.sort_values(by='Quantidade de Contratos',ascending=False)

df_contagem_modalidade.set_index('Modalidade de Compra')

# **Somando todos para obter o valor Total de Contratos**:


total_contratos = df['valorFinalCompra'].sum()
print(f'VALOR TOTAL DE CONTRATOS - R$ {total_contratos:.2f}')

# **Gráfico para visualizar os dados**


import matplotlib.pyplot as plt

modalidades = df_contagem_modalidade['Modalidade de Compra']
quantidades = df_contagem_modalidade['Quantidade de Contratos']

explode = [0.1 if quantidade == max(quantidades) else 0 for quantidade in quantidades]

cores = plt.cm.tab20c.colors  #Paleta de cores pré-definida

plt.figure(figsize=(10, 8))
plt.pie(quantidades, labels=modalidades, autopct='%1.1f%%', startangle=140, explode=explode, colors=cores, shadow=True)

plt.title('Proporção de Contratos por Modalidade de Compra na UNIFESP')

plt.legend(modalidades, title="Modalidades de Compra", loc="center left", bbox_to_anchor=(1, 0, 0.5, 2))

plt.tight_layout()  #Ajusta automaticamente o subplot
plt.show()

