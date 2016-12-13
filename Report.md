## 課題 (スライス機能の拡張)

* スライスの分割・結合
  * スライスの分割と結合機能を追加する
* スライスの可視化
  * ブラウザでスライスの状態を表示
* REST APIの追加
  * 分割・統合のできるAPIを追加


## 解答

###提出者
* 木籐
* 銀杏
* 永富
* 錦織
* 村上


### スライスの分割

[./bin/slice](https://github.com/handai-trema/sliceable-switch-team-w/blob/develop/bin/slice)に変更を加えた。


以下に変更を加えた箇所を示す

```
  desc 'Split a virtual slice'
  arg_name 'name'
  command :split do |c|
    c.desc 'Location to find socket files'
    c.desc 'MAC address'
    c.flag [:m, :mac]
    c.desc 'Switch port'
    c.flag [:p, :port]
    c.desc 'Slice name'
    c.flag [:s, :slice]
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

    c.action do |_global_options, options, args|
      DRb.start_service
      fail 'slice name is required.' if args.empty?
      #slice_new = args[0]
      #slice_a = args[1]
      #slice_b = args[2]
      controller = Trema.trema_process('RoutingSwitch', options[:socket_dir]).controller
      #slice_new = slice(options[:socket_dir]).create(args[0])
      slice_set = nil
      while !(slice_set == 'end') do
        puts "Input slice name= \n"
        slice_set = STDIN.gets.to_s.chomp
        if !(slice_set == 'end') then
          slice_new = slice(options[:socket_dir]).create(slice_set)
          host = nil
          while !(host == 'end') do 
            puts "Input host name= \n"
            host = STDIN.gets.to_s.chomp
            slice(options[:socket_dir]).all.each do |slice|
              if slice.name == args[0] then
                slice.each do |port, mac_addresses|
                  for num in 0 ... mac_addresses.length do
                    if mac_addresses[0] == host then
                      puts "nagatomi"///消す予定
                      slice_new.add_port(dpid: port.fetch(:dpid), port_no: port.fetch(:port_no))
                      slice_new.add_mac_address(host,
                                                dpid: port.fetch(:dpid),
                                                port_no: port.fetch(:port_no))
                    end
                  end
                end
              end
            end
          end
        end
      end
      slice(options[:socket_dir]).destroy(args.first)
      update_slice(options[:socket_dir])
    end
  end
```

####コマンドの実行

以下に示す3つのホストを有するslice5を３つのスライスに分割する。  
```
ensyuu2@ensyuu2-VirtualBox:~/sliceable-switch-team-w$ ./bin/slice list
slice5
  0x4:1
    22:22:22:22:22:22
  0x6:1
    44:44:44:44:44:44
  0x1:1
    11:11:11:11:11:11
```

コマンド入力後、コンソール入力が求められる。引数には分割元のスライスを指定し、新たに分割するスライスの名前→そのスライスに加えるホストのMACアドレスを入力する。
endと入力することでその入力を終了する。
```
ensyuu2@ensyuu2-VirtualBox:~/sliceable-switch-team-w$ ./bin/slice split slice5
Input slice name= 
slice1
Input host name= 
22:22:22:22:22:22
nagatomi///消す予定
Input host name= (MACアドレスを入力)
end

Input slice name= 
slice2
Input host name= 
44:44:44:44:44:44
nagatomi///消す予定
Input host name= 
end

Input slice name= 
slice3
Input host name= 
11:11:11:11:11:11
nagatomi///消す予定
Input host name= 
end(endとしたら、その入力を終了するようになっている。)

Input slice name= 
end
Input host name= 
end

ensyuu2@ensyuu2-VirtualBox:~/sliceable-switch-team-w$ ./bin/slice list
slice1
  0x4:1
    22:22:22:22:22:22
slice2
  0x6:1
    44:44:44:44:44:44
slice3
  0x1:1
    11:11:11:11:11:11

```
最終的にもともと同じスライスにあった3つのホストが別々のスライスに分割されていることがわかる。


###スライスの結合

スライスの分割と同様に、[./bin/slice](https://github.com/handai-trema/sliceable-switch-team-w/blob/develop/bin/slice)に変更を加えた。

変更箇所を以下に示す。複数のスライスを一度に結合できるように実装した。
```
  desc 'Merge a virtual slices'
  arg_name 'name1'
  arg_name 'name', :multiple
  command :merges do |c|
    c.desc 'Location to find socket files'
    #c.switch [:into, :slice_divide]
    c.desc 'MAC address'
    c.flag [:m, :mac]
    c.desc 'Switch port'
    c.flag [:p, :port]
    c.desc 'Slice name'
    c.flag [:s, :slice]
    c.flag [:S, :socket_dir], default_value: Trema::DEFAULT_SOCKET_DIR

    c.action do |_global_options, options, args|
      DRb.start_service
      fail 'slice name is required.' if args.empty?
      controller = Trema.trema_process('RoutingSwitch', options[:socket_dir]).controller
      slice_new = slice(options[:socket_dir]).create(args[0])
      slice(options[:socket_dir]).all.each do |slice|
        for num in 1 ... args.length do
          if slice.name == args[num] then
            slice.each do |port, mac_addresses|
              slice_new.add_port(dpid: port.fetch(:dpid), port_no: port.fetch(:port_no))
              mac_addresses.each do |each|
                slice.delete_mac_address(each,
                                         dpid: port.fetch(:dpid),
                                         port_no: port.fetch(:port_no))
                slice_new.add_mac_address(each,
                                          dpid: port.fetch(:dpid),
                                          port_no: port.fetch(:port_no))
                slice.delete_port(dpid: port.fetch(:dpid), port_no: port.fetch(:port_no))       
              end
            end
          end
        end
      end 
      for num in 1 ... args.length do
        slice(options[:socket_dir]).destroy(args[num])
      end
      update_slice(options[:socket_dir])
    end
  end
```

####コマンドの実行

slice1~slice4の4つのスライスを結合し、新しいslice10というスライスを作成した。

slice merges (結合した後のスライス名) ((結合するスライス）...(結合するスライス))

とコマンドを入力することで実行可能。

```
ensyuu2@ensyuu2-VirtualBox:~/sliceable-switch-team-w$ ./bin/slice list
slice1
  0x1:1
    11:11:11:11:11:11
slice2
  0x4:1
    22:22:22:22:22:22
slice3
  0x5:1
    33:33:33:33:33:33
slice4
  0x6:1
    44:44:44:44:44:44
ensyuu2@ensyuu2-VirtualBox:~/sliceable-switch-team-w$ ./bin/slice merges slice10 slice1 slice2 slice3 slice4
ensyuu2@ensyuu2-VirtualBox:~/sliceable-switch-team-w$ ./bin/slice list
slice10
  0x1:1
    11:11:11:11:11:11
  0x4:1
    22:22:22:22:22:22
  0x5:1
    33:33:33:33:33:33
  0x6:1
    44:44:44:44:44:44

```

以上より、4つのスライスが結合されていることがわかる。
