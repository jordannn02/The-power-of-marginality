# The-power-of-marginality


# 定義 initialize_exp 函式
def initialize_exp():
    # 將 K 利用 Poisson 分布取樣 N 次，並乘以 K 次，並將結果取整數後存入 deg
    deg = [round(i) for i in (np.random.poisson(K, N) * np.random.poisson(K, N)) / K]
    # 將 deg 作為參數傳入 directed_spatial_random_graph，並將回傳結果存入 G_poisson
    G_poisson = directed_spatial_random_graph(
        num_agents=N,
        degree=deg,
        return_positions=False
    )
    # 建立 simrun 物件，並將參數指定為相應之參數
    simrun = Simulation(
        seed=
        network=G_poisson,
        attributes_initializer="random_categorical",  
        dissimilarity_measure="hamming",  
        influence_function="similarity_adoption",  
        neighbor_selector='similar',
        stop_condition="strict_convergence",  
        max_iterations=100000,  # simulation stops after this # of ticks
        communication_regime='one-to-many',
        parameter_dict={  # dictionary for all arguments passed to modules
            'n': N,  # size of the network
            'num_features': 3,  # number of opinion features agents posess
            'num_traits': 3
        })
    # 將 simrun 回傳
    return simrun

# 定義 do_exp 函式
def do_exp(exp_num):
    # 檢查是否已建立對應資料夾，若未建立則建立
    if not os.path.exists(os.path.join(output_dir, str(exp_num))):
        os.makedirs(os.path.join(output_dir, str(exp_num)))
    # 建立該資料夾之網路資訊路徑
    net_info_path = os.path.join(output_dir, str(exp_num), 'network_info.txt')
    # 檢查該路徑是否有網路資訊檔，若有則回傳
    if os.path.exists(net_info_path):
        return
    # 將 initialize_exp 函式回傳結果存入 simrun_
    simrun_ = initialize_exp()
    # 將 simrun_ 傳入 get_net_info 函式，並將回傳結果存入 info
    info = get_net_info(simrun_.network)
    # 將 info 以 json 格式寫入檔案
    with open(net_info_path, 'w') as file:
        file.write(json.dumps(info))  # use `json.loads` to do the reverse
    # 迴圈，執行 rumormonger_type 迴圈
    for rumormonger_type in ['all', 'kol', 'loner']:
        # 建立該資料夾之結果檔路徑
        csv_path = os.path.join(output_dir, str(exp_num), f'{rumormonger_type}_result.csv')
        # 將 simrun_ 複製後存入 simrun
        simrun = copy.deepcopy(simrun_)
        # 執行 simrun 初始化
        simrun.initialize()
        # 建立 result 空字典
        result = {}
        # 將 itertool_ 設定為 ('0', '1', '2'), ('0', '1', '2'), ('0', '1', '2', '99') 的組合
        itertool_ = itertools.product(["0", "1", "2"], ["0", "1", "2"], ["0", "1", "2", "99"])
        # 將 itertool_ 轉為 list 並迴圈，將迴圈結果加入 result 中
        for l in list(itertool_):
            result["Type:" + "_".join(l)] = []
        # 將 simrun 中的網路傳入 get_node_ids 函式，並以 rumormonger_type 篩選出節點，將節點存入 node_ids
        node_ids = get_node_ids(simrun.network, rumormonger_type)
        # 迴圈，執行 steps 次
        for i in range(steps):
            # 將 simrun 中的網路傳入 set_nodes 函式，設定 node_ids 之特性為 {'f01': 1, 'f02': 1, 'f03': 99}
            set_nodes(simrun.network, node_ids=node_ids, values={'f01': 1, 'f02': 1, 'f03': 99})
            # 執行 simrun 單步運行
            simrun.run_step()
            # 再次將 simrun 中的網路傳入 set_nodes 函式，設定 node_ids 之特性為 {'f01': 1, 'f02': 1, 'f03': 99}
            set_nodes(simrun.network, node_ids=node_ids, values={'f01': 1, 'f02': 1, 'f03': 99})
    # 建立結果表格
    df1 = create_output_table(simrun.network, realizations=['Basic', 'RegionsList', 'ClusterFinder'])
    # 將表格結果放入result中
    for key in df1.keys():
        if key not in result.keys():
            result[key] = []
        result[key].append(df1[key])
    # 將Type資訊放入result中
    for key in result.keys():
        if 'Type:' in key:
            result[key].append(0)
    # 將計算節點數量放入result中
    for node_idx in range(len(simrun.network.nodes)):
        key = "Type:" + "_".join([str(simrun.network.nodes[node_idx][key]) for key in simrun.network.nodes[node_idx].keys()])
        result[key][-1] += 1
    # 將result轉換成pandas DataFrame
    result_pd = pd.DataFrame(result)
    # 將DataFrame存成csv檔
    result_pd.to_csv(csv_path)
![image](https://user-images.githubusercontent.com/78151083/207863148-49952689-c922-4d6d-9272-9e519c79ce8f.png)
