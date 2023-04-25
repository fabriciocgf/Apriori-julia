# Apriori-julia
[![DOI](https://zenodo.org/badge/631400645.svg)](https://zenodo.org/badge/latestdoi/631400645)

- Implementação do algoritmo apriori para recomendação de produtos utilizando a linguagemm Julia
- Esta implementação demontra o trabalho realizado para o TCC do MBA em Data science & Analytics da USP

## Funcionamento

o algoritmo foi implementado em duas funções principais: “apriori” e “association_rules”

### Função “apriori”:
```Julia
function apriori(transactions::Vector{Vector{Int64}}, articles::Vector{Int64}, threshold::Float64=0.3, max::Int64=4)
    combination_max_size = max
    combs_of_size = [Set(collect(combinations(articles,n))) for n in collect(1:combination_max_size)]
    transactions_count = length(transactions)
    result = []
    dict_comb = Dict{Array{Int64,1}, Float64}()
    for n in collect(1:combination_max_size)
        for comb in combs_of_size[n]
            found_percentage = sum([issubset(comb,t) for t in transactions]) / transactions_count
            if found_percentage < threshold
                for j in collect(n+1:combination_max_size)
                    if length(combs_of_size[j]) > 0
                        filter!(c -> !issubset(comb, c), combs_of_size[j])
                    end
                end
            else
                dict_comb[comb] = found_percentage
                push!(result, comb)
            end
        end
    end
    return dict_comb, result
end
```

A função “apriori” recebe como entrada uma lista de transações onde cada transação é uma lista de produtos comprados, uma lista de produtos contendo itens disponíveis para compra, um limiar de suporte mínimo e um tamanho máximo de combinação.
A função “apriori” gera um conjunto de combinações de produtos e calcula o suporte de cada combinação na lista de transações.
As combinações que atingem o suporte mínimo são consideradas frequentes e são armazenadas em uma lista de resultados, além disso, o suporte de cada combinação é
armazenado em um dicionário para uso posterior na geração de regras de associação.

### Função “association_rules”:
```Julia
function association_rules(frequent_items_dict::Dict{Vector{Int64}, Float64}, metric::String="confidence", min_threshold::Float64=0.8, min_lift::Float64=0.0)
    metric_dict = Dict{String,Function}(
            "antecedent support" => (_, sA, __) -> sA,
            "consequent support" => (_, __, sC) -> sC,
            "support" => (sAC, _, __) -> sAC,
            "confidence" => (sAC, sA, _) -> sAC ./ sA,
            "lift" => (sAC, sA, sC) -> metric_dict["confidence"](sAC, sA, sC)./sC
    )
    columns_ordered = [
        "antecedent support",
        "consequent support",
        "support",
        "confidence",
        "lift"
		]
    rule_antecedents = []
    rule_consequents = []
    rule_supports = []
    for (k, sAC) in frequent_items_dict
        for idx in collect((length(k)-1):-1:1)
            for c in collect(combinations(k, idx))
                antecedent = collect(c)
                consequent = setdiff(k, antecedent)
      sA = frequent_items_dict[antecedent]
      sC = frequent_items_dict[consequent]
                score = metric_dict[metric](sAC, sA, sC)
                Lift = metric_dict["lift"](sAC, sA, sC)
                if score >= min_threshold && Lift >= min_lift
                        push!(rule_antecedents, antecedent)
                        push!(rule_consequents, consequent)
                        push!(rule_supports, [sAC, sA, sC])
                end
            end
        end
    end
    if isempty(rule_supports)
        return "Error"
    else
        rule_supports = hcat(rule_supports...)
        df_res = DataFrame(antecedents=rule_antecedents, consequents=rule_consequents)
  		sAC = rule_supports[1, :]
  		sA = rule_supports[2, :]
  		sC = rule_supports[3, :]
  		for m in columns_ordered
    		df_res[!, m] = metric_dict[m](sAC, sA, sC)
  		end
        return df_res
    end
end
```

A função “association_rules” recebe como entrada o dicionário de combinações frequentes e um conjunto de parâmetros que especificam a métrica a ser usada para avaliar as regras de associação, bem como o valor mínimo para essa métrica.
A função calcula as regras de associação a partir das combinações frequentes usando a métrica especificada e retorna uma tabela contendo as regras encontradas.

### Função “arl_recommender”:
```Julia
function arl_recommender(rules_df, product_id, rec_count=1)
    sorted_rules = rules_df
    recommendation_list = []
    for i in 1:size(sorted_rules, 1)
        product = sorted_rules[!, "antecedents_str"][i]
        if product in product_id
            push!(recommendation_list, sorted_rules[!, "consequents_str"][i])
        end
    end
    if rec_count > size(product_id, 1)
        rec_count = size(product_id, 1)
    end
    recommendation_list = union(recommendation_list)
    recommendation_list = setdiff(recommendation_list,product_id)
    return recommendation_list[1:rec_count]
end
```

Por fim, a função “arl_recommender” recebe como entrada a tabela de regras de associação, uma lista de produtos que um cliente já comprou e o número de recomendações a serem geradas.
A função filtra as regras que envolvem algum dos produtos comprados pelo cliente e retorna um conjunto de produtos recomendados, com base nos produtos restantes nas regras, que não foram comprados pelo cliente.
