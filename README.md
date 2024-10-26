# Why gRPC client follows its TCP FIN, ACK with RST, ACK?

This behavior occurs inside of a docker container, when container is started directly on a host.
Tested on Ubuntu 20.04 amd64 and MacOS M1 arm64 host.

The behavior does not occur inside of a virtual machine (Ubuntu 22.04 and Debian Bookworm)
started using Vagrant on Ubuntu 20.04 amd64 host, neither inside of a docker container started
inside of these virtual machines.

The behavior occurs when the code is executed directly on the Ubuntu 20.04 amd64 host,
and does not occur when the code is executed directly on MacOS M1 arm64 host.

Since the behavior appears not repeatable, don't interpret the above "occurs" and "does not occur" as "always" and "never". 

The question asked at https://groups.google.com/g/grpc-io/c/r1S8xkncQTQ

# Answer

?

# Reproduce the behavior inside of a docker container.

0. Set the gRPC version.
   ```sh
   GRPC_VERSION=1.67.0
   ```

1. Clone the gRPC quickstarts.
   ```sh
   # https://grpc.io/docs/languages/python/quickstart/
   git clone -b v${GRPC_VERSION} --depth 1 --shallow-submodules https://github.com/grpc/grpc
   # https://grpc.io/docs/languages/go/quickstart/
   git clone -b v${GRPC_VERSION} --depth 1 https://github.com/grpc/grpc-go
   ```

2. Start a container. Add `--privileged` if planning to use iptables.
   ```sh
   docker run --detach --rm -v $PWD/grpc:/opt/grpc -v $PWD/grpc-go:/opt/grpc-go --name grpc debian:unstable bash -c "sleep infinity"
   docker exec -it grpc bash -c "uname -a && tshark --version"
   docker --version

   # Linux 79ae7d0391cd 6.6.51-0-virt #1-Alpine SMP PREEMPT_DYNAMIC 2024-09-12 12:56:22 aarch64 GNU/Linux
   # TShark (Wireshark) 4.4.1.
   # Docker version 27.2.1-rd, build cc0ee3e
   ```

3. Install packages.
   ```sh
   docker exec -it grpc bash -c "apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install --yes procps vim curl golang-go iproute2 iptables jq python3 python3-pip python3-venv tcpdump wireshark-common tshark"
   docker exec -it grpc bash -c "curl -sL https://api.github.com/repos/fullstorydev/grpcurl/releases/latest | jq -r '.assets[] | .browser_download_url' | grep deb | grep -E "$(uname -m | sed 's/x86_64/amd64/;s/aarch64/arm64/')" | tee /dev/tty | xargs curl -sL -o grpcurl.deb && dpkg -i grpcurl.deb && rm -f grpcurl.deb && which grpcurl && grpcurl --version"
   docker exec -it grpc bash -c "python3 -m venv venv && . venv/bin/activate && python -m pip install grpcio==${GRPC_VERSION} protobuf"
   # https://github.com/grpc/grpc/issues/37610
   docker exec -it grpc bash -c "sed -i 's/1.67.0.dev0/1.67.0/' /opt/grpc/examples/python/helloworld/helloworld_pb2_grpc.py"
   ```

4. A Python gRPC client always (?) follows its TCP FIN, ACK with RST, ACK.
   ```sh
   # 4a. In the first terminal, start server.
   docker exec -it grpc bash -c ". venv/bin/activate && python /opt/grpc/examples/python/helloworld/greeter_server.py"

   # 4b. In the second, start tcdump.
   docker exec -it grpc bash -c "tcpdump -i lo -w /tmp/python_python.pcap 'port 50051' && cp /tmp/python_python.pcap /opt/grpc"

   # 4c. In the third terminal, start client.
   docker exec -it grpc bash -c ". venv/bin/activate && GRPC_TRACE=all GRPC_VERBOSITY=DEBUG python /opt/grpc/examples/python/helloworld/greeter_client.py"

   # Will try to greet world ...
   # Greeter client received: Hello, you!
   # The verbose.log is included in the repo as a separate file, similarly to the pcap files.

   # 4d. Stop tcdump with ctrl+c, and read the packet capture.
   docker exec -it grpc sh -c "tshark -r /opt/grpc/python_python.pcap"

   #    1   0.000000   ::1 ? ::1   TCP 94 45118 ? 50051 [SYN] Seq=0 Win=65476 Len=0 MSS=65476 SACK_PERM TSval=2697583267 TSecr=0 WS=128
   #    2   0.000019   ::1 ? ::1   TCP 94 50051 ? 45118 [SYN, ACK] Seq=0 Ack=1 Win=65464 Len=0 MSS=65476 SACK_PERM TSval=2697583267 TSecr=2697583267 WS=128
   #    3   0.000040   ::1 ? ::1   TCP 86 45118 ? 50051 [ACK] Seq=1 Ack=1 Win=65536 Len=0 TSval=2697583267 TSecr=2697583267
   #    4   0.000533   ::1 ? ::1   HTTP2 168 Magic, SETTINGS[0], WINDOW_UPDATE[0]
   #    5   0.000537   ::1 ? ::1   HTTP2 132 SETTINGS[0], WINDOW_UPDATE[0]
   #    6   0.000553   ::1 ? ::1   TCP 86 45118 ? 50051 [ACK] Seq=83 Ack=47 Win=65536 Len=0 TSval=2697583267 TSecr=2697583267
   #    7   0.000555   ::1 ? ::1   TCP 86 50051 ? 45118 [ACK] Seq=47 Ack=83 Win=65536 Len=0 TSval=2697583268 TSecr=2697583267
   #    8   0.000663   ::1 ? ::1   HTTP2 95 SETTINGS[0]
   #    9   0.000665   ::1 ? ::1   TCP 86 45118 ? 50051 [ACK] Seq=83 Ack=56 Win=65536 Len=0 TSval=2697583268 TSecr=2697583268
   #   10   0.001009   ::1 ? ::1   GRPCHTTP2 366 SETTINGS[0], HEADERS[1]: POST /helloworld.Greeter/SayHello, WINDOW_UPDATE[1], DATA[1] (GRPC) (PROTOBUF), WIND#OW_UPDATE[0]
   #   11   0.001188   ::1 ? ::1   HTTP2 103 PING[0]
   #   12   0.001226   ::1 ? ::1   HTTP2 103 PING[0]
   #   13   0.001990   ::1 ? ::1   GRPCHTTP2 252 HEADERS[1]: 200 OK, DATA[1] (GRPC) (PROTOBUF), HEADERS[1], WINDOW_UPDATE[0]
   #   14   0.002156   ::1 ? ::1   HTTP2 103 PING[0] # client -> server
   #   15   0.002202   ::1 ? ::1   HTTP2 103 PING[0] # server -> client
   #   16   0.002406   ::1 ? ::1   TCP 86 45118 ? 50051 [FIN, ACK] Seq=397 Ack=256 Win=65536 Len=0 TSval=2697583269 TSecr=2697583269
   #   17   0.002428   ::1 ? ::1   TCP 86 45118 ? 50051 [RST, ACK] Seq=398 Ack=256 Win=65536 Len=0 TSval=2697583269 TSecr=2697583269
   ```

5. A Golang gRPC client sometimes follows its TCP FIN, ACK with RST, ACK, but often uses the usual FIN -> FIN -> ACK sequence.
   ```sh
   # 5a. In the first terminal, start server.
   docker exec -it grpc bash -c "cd /opt/grpc-go/examples/helloworld && go run greeter_server/main.go"

   # 5b. In the second, start tcdump.
   docker exec -it grpc bash -c "tcpdump -i lo -w /tmp/golang_golang.pcap 'port 50051' && cp /tmp/golang_golang.pcap /opt/grpc-go"

   # 5c. In the third terminal, start client.
   docker exec -it grpc bash -c "cd /opt/grpc-go/examples/helloworld && go run greeter_client/main.go"

   # 2024/10/20 16:24:04 Greeting: Hello world

   # 5d. Stop tcdump with ctrl+c, and read the packet capture.
   docker exec -it grpc sh -c "tshark -r /opt/grpc-go/golang_golang.pcap"

   #    1   0.000000    127.0.0.1 ? 127.0.0.1    TCP 74 42136 ? 50051 [SYN] Seq=0 Win=65495 Len=0 MSS=65495 SACK_PERM TSval=2409056769 TSecr=0 WS=128
   #    2   0.000030    127.0.0.1 ? 127.0.0.1    TCP 74 50051 ? 42136 [SYN, ACK] Seq=0 Ack=1 Win=65483 Len=0 MSS=65495 SACK_PERM TSval=2409056769 TSecr=2409056769 WS=128
   #    3   0.000051    127.0.0.1 ? 127.0.0.1    TCP 66 42136 ? 50051 [ACK] Seq=1 Ack=1 Win=65536 Len=0 TSval=2409056769 TSecr=2409056769
   #    4   0.000688    127.0.0.1 ? 127.0.0.1    TCP 81 50051 ? 42136 [PSH, ACK] Seq=1 Ack=1 Win=65536 Len=15 TSval=2409056770 TSecr=2409056769
   #    5   0.000714    127.0.0.1 ? 127.0.0.1    HTTP2 90 Magic
   #    6   0.000773    127.0.0.1 ? 127.0.0.1    TCP 66 50051 ? 42136 [ACK] Seq=16 Ack=25 Win=65536 Len=0 TSval=2409056770 TSecr=2409056770
   #    7   0.000792    127.0.0.1 ? 127.0.0.1    TCP 66 42136 ? 50051 [ACK] Seq=25 Ack=16 Win=65536 Len=0 TSval=2409056770 TSecr=2409056770
   #    8   0.000869    127.0.0.1 ? 127.0.0.1    HTTP2 75 SETTINGS[0]
   #    9   0.000875    127.0.0.1 ? 127.0.0.1    TCP 66 50051 ? 42136 [ACK] Seq=16 Ack=34 Win=65536 Len=0 TSval=2409056770 TSecr=2409056770
   #   10   0.001214    127.0.0.1 ? 127.0.0.1    HTTP2 75 SETTINGS[0]
   #   11   0.001249    127.0.0.1 ? 127.0.0.1    HTTP2 175 SETTINGS[0], HEADERS[1]: POST /helloworld.Greeter/SayHello
   #   12   0.001317    127.0.0.1 ? 127.0.0.1    GRPC/PB(<UNKNOWN>) 87 DATA[1] (GRPC) (PROTOBUF)
   #   13   0.001365    127.0.0.1 ? 127.0.0.1    TCP 66 50051 ? 42136 [ACK] Seq=25 Ack=164 Win=65536 Len=0 TSval=2409056771 TSecr=2409056771
   #   14   0.001764    127.0.0.1 ? 127.0.0.1    GRPCHTTP2 179 WINDOW_UPDATE[0], PING[0], HEADERS[1]: 200 OK, DATA[1] (GRPC) (PROTOBUF), HEADERS[1]
   #   15   0.002306    127.0.0.1 ? 127.0.0.1    HTTP2 113 PING[0], WINDOW_UPDATE[0], PING[0]
   #   16   0.002380    127.0.0.1 ? 127.0.0.1    HTTP2 83 PING[0]
   #   17   0.002408    127.0.0.1 ? 127.0.0.1    HTTP2 108 GOAWAY[0]
   #   18   0.002455    127.0.0.1 ? 127.0.0.1    TCP 66 42136 ? 50051 [FIN, ACK] Seq=253 Ack=155 Win=65536 Len=0 TSval=2409056772 TSecr=2409056772
   #   19   0.002489    127.0.0.1 ? 127.0.0.1    TCP 66 50051 ? 42136 [FIN, ACK] Seq=155 Ack=254 Win=65536 Len=0 TSval=2409056772 TSecr=2409056772
   #   20   0.002677    127.0.0.1 ? 127.0.0.1    TCP 66 42136 ? 50051 [ACK] Seq=254 Ack=156 Win=65536 Len=0 TSval=2409056772 TSecr=2409056772
   ```

6. A grpcurl gRPC client sometimes follows its TCP FIN, ACK with RST, ACK, but often uses the usual FIN -> FIN -> ACK sequence.
   ```sh
   docker exec -it grpc bash -c "grpcurl -import-path /opt/grpc/examples/protos -proto helloworld.proto -plaintext -d '{ \"name\": \"you\" }' localhost:50051 helloworld.Greeter/SayHello"
   # {
   #   "message": "Hello, you!"
   # }
   ```

7. Frequency of TCP RST present during Golang gRPC use can be estimated using a loop.
Requires https://www.gnu.org/software/screen/manual/screen.html installed on the host.
   ```sh
   # 7a. In the first terminal, start server.
   docker exec -it grpc bash -c "cd /opt/grpc-go/examples/helloworld && go run greeter_server/main.go"

   # 7b. In the second terminal, perform requsts in a loop. Stop with ctrl+c.
   while true;
   do
   date -u &&
   echo "ss -ant" && docker exec -it grpc bash -c "ss -ant"
   (docker exec -it grpc bash -c 'kill -SIGINT $(pgrep --full "tcpdump")' || true
   screen -S tcpdump -d -m bash -c "docker exec -it grpc bash -c 'rm -f /tmp/test.pcap && tcpdump -i lo -w /tmp/test.pcap port 50051'"
   sleep 2
   docker exec -it grpc bash -c "cd /opt/grpc-go/examples/helloworld && go run greeter_client/main.go"
   sleep 2
   screen -ls
   docker exec -it grpc bash -c 'kill -SIGINT $(pgrep --full "tcpdump")' || true
   sleep 1
   docker exec -it grpc bash -c "rm -f /opt/grpc/test.pcap && cp /tmp/test.pcap /opt/grpc"
   screen -X -S "tcpdump" quit) > /dev/null 2>&1
   echo "tshark -r /opt/grpc/test.pcap"
   docker exec -it grpc bash -c "tshark -r /opt/grpc/test.pcap 2> /dev/null | grep -B 2 -A 1 RST || echo No RST" 
   done

   # Fri Oct 18 16:53:53 UTC 2024
   # ss -ant
   # State                Recv-Q               Send-Q                             Local Address:Port                                Peer Address:Port               
   # LISTEN               0                    4096                                           *:50051                                          *:*                  
   # tshark -r /opt/grpc/test.pcap
   #    18   0.004477    127.0.0.1 → 127.0.0.1    HTTP2 108 GOAWAY[0]
   #    19   0.004487    127.0.0.1 → 127.0.0.1    HTTP2 83 PING[0]
   #    20   0.004508    127.0.0.1 → 127.0.0.1    TCP 66 49794 → 50051 [RST, ACK] Seq=253 Ack=155 Win=65536 Len=0 TSval=2410450919 TSecr=2410450919
   # Fri Oct 18 16:54:00 UTC 2024
   # ss -ant
   # State                Recv-Q               Send-Q                             Local Address:Port                                Peer Address:Port               
   # LISTEN               0                    4096                                           *:50051                                          *:*                  
   # tshark -r /opt/grpc/test.pcap
   # No RST
   # Fri Oct 18 16:54:07 UTC 2024
   # ss -ant
   # State                  Recv-Q               Send-Q                             Local Address:Port                              Peer Address:Port               
   # TIME-WAIT              0                    0                                      127.0.0.1:50950                                127.0.0.1:50051              
   # LISTEN                 0                    4096                                           *:50051                                        *:*                  
   # tshark -r /opt/grpc/test.pcap
   # No RST
   # Fri Oct 18 16:54:13 UTC 2024
   # ss -ant
   # State                  Recv-Q               Send-Q                             Local Address:Port                              Peer Address:Port               
   # TIME-WAIT              0                    0                                      127.0.0.1:50956                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:50950                                127.0.0.1:50051              
   # LISTEN                 0                    4096                                           *:50051                                        *:*                  
   # tshark -r /opt/grpc/test.pcap
   # No RST
   # Fri Oct 18 16:54:20 UTC 2024
   # ss -ant
   # State                  Recv-Q               Send-Q                             Local Address:Port                              Peer Address:Port               
   # TIME-WAIT              0                    0                                      127.0.0.1:46398                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:50956                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:50950                                127.0.0.1:50051              
   # LISTEN                 0                    4096                                           *:50051                                        *:*                  
   # tshark -r /opt/grpc/test.pcap
   # No RST
   # Fri Oct 18 16:54:26 UTC 2024
   # ss -ant
   # State                  Recv-Q               Send-Q                             Local Address:Port                              Peer Address:Port               
   # TIME-WAIT              0                    0                                      127.0.0.1:46398                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:50956                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:46406                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:50950                                127.0.0.1:50051              
   # LISTEN                 0                    4096                                           *:50051                                        *:*                  
   # tshark -r /opt/grpc/test.pcap
   # No RST
   # Fri Oct 18 16:54:32 UTC 2024
   # ss -ant
   # State                  Recv-Q               Send-Q                             Local Address:Port                              Peer Address:Port               
   # TIME-WAIT              0                    0                                      127.0.0.1:37652                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:46398                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:50956                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:46406                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:50950                                127.0.0.1:50051              
   # LISTEN                 0                    4096                                           *:50051                                        *:*                  
   # tshark -r /opt/grpc/test.pcap
   # No RST
   # Fri Oct 18 16:54:39 UTC 2024
   # ss -ant
   # State                  Recv-Q               Send-Q                             Local Address:Port                              Peer Address:Port               
   # TIME-WAIT              0                    0                                      127.0.0.1:37652                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:46398                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:50956                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:46406                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:34250                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:50950                                127.0.0.1:50051              
   # LISTEN                 0                    4096                                           *:50051                                        *:*                  
   # tshark -r /opt/grpc/test.pcap
   # No RST
   # Fri Oct 18 16:54:45 UTC 2024
   # ss -ant
   # State                  Recv-Q               Send-Q                             Local Address:Port                              Peer Address:Port               
   # TIME-WAIT              0                    0                                      127.0.0.1:37652                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:46398                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:50956                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:46406                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:34250                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:34252                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:50950                                127.0.0.1:50051              
   # LISTEN                 0                    4096                                           *:50051                                        *:*                  
   # tshark -r /opt/grpc/test.pcap
   # No RST
   # Fri Oct 18 16:54:52 UTC 2024
   # ss -ant
   # State                  Recv-Q               Send-Q                             Local Address:Port                              Peer Address:Port               
   # TIME-WAIT              0                    0                                      127.0.0.1:37652                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:46398                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:50956                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:46406                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:34250                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:34252                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:55704                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:50950                                127.0.0.1:50051              
   # LISTEN                 0                    4096                                           *:50051                                        *:*                  
   # tshark -r /opt/grpc/test.pcap
   # No RST
   # Fri Oct 18 16:54:58 UTC 2024
   # ss -ant
   # State                  Recv-Q               Send-Q                             Local Address:Port                              Peer Address:Port               
   # TIME-WAIT              0                    0                                      127.0.0.1:37652                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:46398                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:50956                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:42286                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:46406                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:34250                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:34252                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:55704                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:50950                                127.0.0.1:50051              
   # LISTEN                 0                    4096                                           *:50051                                        *:*                  
   # tshark -r /opt/grpc/test.pcap
   # No RST
   # Fri Oct 18 16:55:05 UTC 2024
   # ss -ant
   # State                  Recv-Q               Send-Q                             Local Address:Port                              Peer Address:Port               
   # TIME-WAIT              0                    0                                      127.0.0.1:37652                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:46398                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:50956                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:42286                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:46406                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:34250                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:34252                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:55704                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:42292                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:50950                                127.0.0.1:50051              
   # LISTEN                 0                    4096                                           *:50051                                        *:*                  
   # tshark -r /opt/grpc/test.pcap
   # No RST
   # Fri Oct 18 16:55:11 UTC 2024
   # ss -ant
   # State                  Recv-Q               Send-Q                             Local Address:Port                              Peer Address:Port               
   # TIME-WAIT              0                    0                                      127.0.0.1:37652                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:46398                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:53614                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:50956                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:42286                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:46406                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:34250                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:34252                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:55704                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:42292                                127.0.0.1:50051              
   # LISTEN                 0                    4096                                           *:50051                                        *:*                  
   # tshark -r /opt/grpc/test.pcap
   #    17   0.002380    127.0.0.1 → 127.0.0.1    HTTP2 83 PING[0]
   #    18   0.002382    127.0.0.1 → 127.0.0.1    HTTP2 108 GOAWAY[0]
   #    19   0.002435    127.0.0.1 → 127.0.0.1    TCP 66 41800 → 50051 [RST, ACK] Seq=254 Ack=155 Win=65536 Len=0 TSval=2410528647 TSecr=2410528647
   # Fri Oct 18 16:55:18 UTC 2024
   # ss -ant
   # State                  Recv-Q               Send-Q                             Local Address:Port                              Peer Address:Port               
   # TIME-WAIT              0                    0                                      127.0.0.1:37652                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:53614                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:42286                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:46406                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:34250                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:34252                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:55704                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:42292                                127.0.0.1:50051              
   # LISTEN                 0                    4096                                           *:50051                                        *:*                  
   # tshark -r /opt/grpc/test.pcap
   # No RST
   # Fri Oct 18 16:55:24 UTC 2024
   # ss -ant
   # State                  Recv-Q               Send-Q                             Local Address:Port                              Peer Address:Port               
   # TIME-WAIT              0                    0                                      127.0.0.1:37652                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:53614                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:42286                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:46406                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:34250                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:34252                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:55704                                127.0.0.1:50051              
   # TIME-WAIT              0                    0                                      127.0.0.1:42292                                127.0.0.1:50051              
   # LISTEN                 0                    4096                                           *:50051                                        *:*                  
   # tshark -r /opt/grpc/test.pcap
   #    16   0.002557    127.0.0.1 → 127.0.0.1    TCP 66 36446 → 50051 [FIN, ACK] Seq=254 Ack=138 Win=65536 Len=0 TSval=2410743678 TSecr=2410743677
   #    17   0.002572    127.0.0.1 → 127.0.0.1    HTTP2 83 PING[0] # server -> client
   #    18   0.002591    127.0.0.1 → 127.0.0.1    TCP 54 36446 → 50051 [RST] Seq=255 Win=0 Len=0
   ```

# No reproduction in virtual machines

8. Start the virtual machines.
   ```sh
   vagrant up
   vagrant ssh vm1 -c "python3 -m venv venv && . venv/bin/activate && python -m pip install grpcio==${GRPC_VERSION} protobuf"
   vagrant ssh vm2 -c "python3 -m venv venv && . venv/bin/activate && python -m pip install grpcio==${GRPC_VERSION} protobuf"
   vagrant ssh vm1 -c "sed -i 's/1.67.0.dev0/1.67.0/' /vagrant/grpc/examples/python/helloworld/helloworld_pb2_grpc.py"
   ```

9. A Python gRPC client uses the expected FIN -> FIN -> ACK TCP connection termination.
   ```sh
   # 9a. In the first terminal, start server.
   vagrant ssh vm1 -c ". venv/bin/activate && python /vagrant/grpc/examples/python/helloworld/greeter_server.py"

   # 9b. In the second, start tcdump.
   vagrant ssh vm1 -c "sudo tcpdump -i lo -w /tmp/python_python_vagrant.pcap 'port 50051' && cp /tmp/python_python_vagrant.pcap /vagrant/grpc"

   # 9c. In the third terminal, start client.
   vagrant ssh vm1 -c ". venv/bin/activate && GRPC_TRACE=all GRPC_VERBOSITY=DEBUG python /vagrant/grpc/examples/python/helloworld/greeter_client.py"

   # Will try to greet world ...
   # Greeter client received: Hello, you!
   # The verbose_vagrant.log is included in the repo as a separate file, similarly to the pcap files.

   # 9d. Stop tcdump with ctrl+c, and read the packet capture.
   vagrant ssh vm1 -c "tshark -r /vagrant/grpc/python_python_vagrant.pcap"

   #    1   0.000000   ::1 ? ::1   TCP 94 45864 ? 50051 [SYN] Seq=0 Win=65476 Len=0 MSS=65476 SACK_PERM TSval=1997884276 TSecr=0 WS=128
   #    2   0.000033   ::1 ? ::1   TCP 94 50051 ? 45864 [SYN, ACK] Seq=0 Ack=1 Win=65464 Len=0 MSS=65476 SACK_PERM TSval=1997884276 TSecr=1997884276 WS=128
   #    3   0.000051   ::1 ? ::1   TCP 86 45864 ? 50051 [ACK] Seq=1 Ack=1 Win=65536 Len=0 TSval=1997884276 TSecr=1997884276
   #    4   0.000669   ::1 ? ::1   TCP 132 50051 ? 45864 [PSH, ACK] Seq=1 Ack=1 Win=65536 Len=46 TSval=1997884277 TSecr=1997884276
   #    5   0.000860   ::1 ? ::1   TCP 86 45864 ? 50051 [ACK] Seq=1 Ack=47 Win=65536 Len=0 TSval=1997884277 TSecr=1997884277
   #    6   0.003955   ::1 ? ::1   HTTP2 168 Magic, SETTINGS[0], WINDOW_UPDATE[0]
   #    7   0.003969   ::1 ? ::1   TCP 86 50051 ? 45864 [ACK] Seq=47 Ack=83 Win=65536 Len=0 TSval=1997884280 TSecr=1997884280
   #    8   0.004100   ::1 ? ::1   HTTP2 95 SETTINGS[0]
   #    9   0.011258   ::1 ? ::1   GRPCHTTP2 366 SETTINGS[0], HEADERS[1]: POST /helloworld.Greeter/SayHello, WINDOW_UPDATE[1], DATA[1] (GRPC) (PROTOBUF), WINDOW_UPDATE[0]
   #   10   0.011402   ::1 ? ::1   HTTP2 103 PING[0]
   #   11   0.013222   ::1 ? ::1   HTTP2 103 PING[0]
   #   12   0.019207   ::1 ? ::1   GRPCHTTP2 252 HEADERS[1]: 200 OK, DATA[1] (GRPC) (PROTOBUF), HEADERS[1], WINDOW_UPDATE[0]
   #   13   0.023496   ::1 ? ::1   HTTP2 103 PING[0]
   #   14   0.023593   ::1 ? ::1   HTTP2 103 PING[0]
   #   15   0.026607   ::1 ? ::1   TCP 86 45864 ? 50051 [FIN, ACK] Seq=397 Ack=256 Win=65536 Len=0 TSval=1997884303 TSecr=1997884300
   #   16   0.026753   ::1 ? ::1   TCP 86 50051 ? 45864 [FIN, ACK] Seq=256 Ack=398 Win=65536 Len=0 TSval=1997884303 TSecr=1997884303
   #   17   0.026764   ::1 ? ::1   TCP 86 45864 ? 50051 [ACK] Seq=398 Ack=257 Win=65536 Len=0 TSval=1997884303 TSecr=1997884303
   ```

10. Perform steps 0. - 4. with docker, inside of `vagrant ssh vm1`. The expected FIN -> FIN -> ACK TCP connection termination occurs.
