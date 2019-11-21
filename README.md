# fabric_data_privacy

■ 참조 사이트 : https://medium.com/@kctheservant/data-privacy-among-organizations-channel-and-private-data-in-hyperledger-fabric-ee268cd44916
                https://github.com/kctam/dataprivacy

< Data Privacy among Organizations: Channel and Private Data in Hyperledger Fabric >

1. 개요
- 승인되지 않은 조직이 개인정보 데이터에 액세스하지 못하도록하거나, 권한이없는 조직이 특정 데이터에 액세스하지 않도록하는 방법에 대해서 진행합니다.
- Hyperledger Fabric에는 두 가지 방법 : 네트워크 레벨에서 채널 사용 및 체인 코드(애플리케이션) 레벨에서 개인 데이터 사용 방법 두 가지를 제시합니다.

2. Consortium 구성 (그림참조)
- A consortium of three organizations : org1, org2 and org3
- 각 조직내에 peer node(peer0) with couchdb

3. Channel
- 컨소시엄에서 필요한만큼 채널을 정의 할 수 있습니다.
- 조직은 비즈니스 요구 사항에 따라 하나 이상의 채널에 참여할 수 있습니다.
- 채널은 채널 ID로 식별되며 회원 조직으로 정의됩니다. 피어가 채널에 참여한 후, 원장(composed of a blockchain of transaction records and a world state database)이 만들어지고 피어에서 실행됩니다.

4. Demo 시나리오 및 Setup (two channel 생성)
- channel-all : org1, org2 및 org3
- channel-12: org1 and org2

4.1 Setup Scripts 내용
- network-up.sh : docker composer to bring up the components (containers)
- network-down.sh : tear down everything.
- channel-all-up.sh and channel-12-up.sh : create channel and join those member organizations to the channel.
- deploy-sacc.sh and deploy-personalinfo.sh : install and instantiate the chaincode

$ cd fabric-samples
$ git clone https://github.com/kctam/dataprivacy.git
$ cd dataprivacy/network

//In case you are using Release 1.4.3
//### in configtx.html line 79, change 
//### from V1_4_2: true 
//### to V1_4_3: true

1 단계 bring up all containers : ./network-up.sh
2 단계 bring up channel-all : ./channel-all-up.sh
3 단계 bring up channel-12 : ./channel-12-up.sh
4 단계 deploy sacc on both channels : ./deploy-sacc.sh
5 단계 clean up : ./network-down.sh

4.2 Demo and Observation
- 1–4 단계를 수행
- deploy-sacc.sh 주의 사항 : initial value가 channel-all [“name”:”alice”], channel-12 [“name”:”bob”] 상이함.

4.2.1 채널 검사
- peer node :  peer0.org1.example.com,  peer0.org2.example.com,  peer0.org3.example.com

$ docker exec peer0.org1.example.com peer channel list
- Result : Channels peers has joined: channel-all, channel-12

$ docker exec peer0.org2.example.com peer channel list
- Result : Channels peers has joined: channel-all, channel-12

$ docker exec peer0.org3.example.com peer channel list
- Result : Channels peers has joined: channel-all

4.2.2 두개의 채널에서 org1으로부터 name의 value 가져오기
$ docker exec cli peer chaincode query -C channel-all -n mycc -c '{"Args":["get","name"]}'
- Result : alice

$ docker exec cli peer chaincode query -C channel-12 -n mycc -c '{"Args":["get","name"]}'
- Result : bob

4.2.3 두개의 채널에서 org3으로부터 name의 value 가져오기

4.2.3.1 channel-all
$ docker exec -e CORE_PEER_LOCALMSPID=Org3MSP -e CORE_PEER_ADDRESS=peer0.org3.example.com:7051 -e CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/users/Admin@org3.example.com/msp cli peer chaincode query -C channel-all -n mycc -c '{"Args":["get","name"]}'
- Result : alice

4.2.3.2 channel-12
- org3은 channel-all 의 멤버 이지만 channel-12 는 아니기 때문 아래 Error 발생함.

$ docker exec -e CORE_PEER_LOCALMSPID=Org3MSP -e CORE_PEER_ADDRESS=peer0.org3.example.com:7051 -e CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/users/Admin@org3.example.com/msp cli peer chaincode query -C channel-12 -n mycc -c '{"Args":["get","name"]}'
- Result : Error: error endorsing query: rpc error: code = Unknown desc = access denied:

4.2.4 두개의 채널에 가입된 peer0.org1.example.com에서 peer가 원장을 핸들링하는 모습 (그림참조)
- 블록 체인 높이와 해시가 다른 두 개(각 채널별)의 별도 블록 체인이 존재함.
- peer는 채널당 각각의 ledger = blockchain in ledger + world state in ledger를 가지고 있음.

$ docker exec peer0.org1.example.com peer channel getinfo -c channel-all
- Result : Blockchain info: {"height":5,"currentBlockHash":"J1Z5C2687W0F7cheXbsOqXrPle9ah8HhNiGiEgqjn+4=","previousBlockHash":"K1HnditRyvltPzA1lSjvSyWLRxeiHODIyqNrZkPjn2Q="}

$ docker exec peer0.org1.example.com peer channel getinfo -c channel-12
- Result : Blockchain info: {"height":2,"currentBlockHash":"jAQ+ft3RIR4XWnCOq9aKf9u8O23+cLCyvIXBEo5ezwA=","previousBlockHash":"Z8f0w3D04bLfrBfxdAei6sf9ZXGOkJQHUE5mMjtsdXo="}

4.2.4.1 world state
- http://localhost:5984/_utils/
- 동일한 couchdb 노드에 있지만 두 채널은 두 개의 개별 데이터베이스 (channel-all_ * 및 channel-12_ *)로 표시됩니다.

5. Private Data (보험 청구 사례)
- channel for all participants 구성
- a separate one for clinics keeping their client information
- Private Data can simplify the whole process with one channel.

5.1 Demo 시나리오 및 체인정보
- all organizations join channel-all.
- chaincode : personalinfo (name, email and passport number)
- passport number : personal identifiable information (PII) => Org1 and Org2에만 접근 가능한 정보

5.1.1 Three chaincode functions 및 data 접근제어 파일
- createRecord : 여권 번호를 포함하여 제공된 개인 정보가있는 레코드를 생성
- queryRecord  : PII를 제외한 개인 레코드를 리턴
- queryPiiRecord : 개인 기록의 PII를 리턴
- the privacy of data 지정하기 위한 파일 collections_config.json : collectionPrivate는 org1 및 org2 에 의해서만 액세스 가능함 (파일의 내용을 참조하세요)
- collections_config.json는 체인 코드가 인스턴스화 될 때 지정됨.

5.2 Demo Setup
- 1 단계 bring up all containers : ./network-up.sh
- 2 단계 bring up channel-all : ./channel-all-up.sh
- 3 단계 deploy personalinfo : ./deploy-personalinfo.sh
- 4 단계 chaincode invoke/query and observation
- 5 단계 clean up: ./network-down.sh

5.3 Demo and Observation
- 위의 1 ~ 3 단계를 수행

5.3.1 chaincode invoke (peer0.org1 node)
- peer0.org1 노드(기본 노드)에서 작업하고 새로운 개인 정보 레코드를 생성.
$ docker exec cli peer chaincode invoke -o orderer.example.com:7050 -C channel-all -n mycc -c '{"Args":["createRecord","ID001","alice","alice@alice.com","H123456"]}'
- Result : Chaincode invoke successful. result: status:200

5.3.2 chaincode query (peer0.org1 node)
- peer0.org1 node : 두 개의 서로 다른 체인 코드 함수로 레코드를 쿼리

$ docker exec cli peer chaincode query -C channel-all -n mycc -c '{"Args":["queryRecord","ID001"]}'
- Result : {"email":"alice@alice.com","id":"ID001","name":"alice"}

$ docker exec cli peer chaincode query -C channel-all -n mycc -c '{"Args":["queryPiiRecord","ID001"]}'
- Result : {"id":"ID001","passport":"H123456"}

5.3.3 chaincode query (peer0.org2 node)
- peer0.org2 node : 두 개의 서로 다른 체인 코드 함수로 레코드를 쿼리

$ docker exec -e CORE_PEER_LOCALMSPID=Org2MSP -e CORE_PEER_ADDRESS=peer0.org2.example.com:7051 -e CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp cli peer chaincode query -C channel-all -n mycc -c '{"Args":["queryRecord","ID001"]}'
- Result : {"email":"alice@alice.com","id":"ID001","name":"alice"}

$ docker exec -e CORE_PEER_LOCALMSPID=Org2MSP -e CORE_PEER_ADDRESS=peer0.org2.example.com:7051 -e CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp cli peer chaincode query -C channel-all -n mycc -c '{"Args":["queryPiiRecord","ID001"]}'
- Result : {"id":"ID001","passport":"H123456"}

5.3.3 chaincode query (peer0.org3 node)
- peer0.org3 node : 두 개의 서로 다른 체인 코드 함수로 레코드를 쿼리
$ docker exec -e CORE_PEER_LOCALMSPID=Org3MSP -e CORE_PEER_ADDRESS=peer0.org3.example.com:7051 -e CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/users/Admin@org3.example.com/msp cli peer chaincode query -C channel-all -n mycc -c '{"Args":["queryRecord","ID001"]}'
- Result : {"email":"alice@alice.com","id":"ID001","name":"alice"}

$ docker exec -e CORE_PEER_LOCALMSPID=Org3MSP -e CORE_PEER_ADDRESS=peer0.org3.example.com:7051 -e CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/users/Admin@org3.example.com/msp cli peer chaincode query -C channel-all -n mycc -c '{"Args":["queryPiiRecord","ID001"]}'
- Result : Error: endorsement failure during query. response: status:500 message:"GET_STATE failed: transaction ID: 83f99378189fff11fce5f7e058834b8b1b8c967a9cd82eee942427219e3845e2: private data matching public hash version is not available. Public hash version = {BlockNum: 5, TxNum: 0}, Private data version = <nil>"
- private data matching public hash version is not available. Public hash version = {BlockNum: 5, TxNum: 0}, Private data version = <nil>" 의미 : 개인 데이터 해시는 공용 저장소에 있지만 실제 해당 데이터는 개인 저장소에 없음을 의미.

5.3.4 the world state on all peer nodes (DB 그림참조 : Edited by combining the couchdb of org1, org2 and org3, respectively.)
- http://localhost:5984/_utils/
- http://localhost:6984/_utils/
- http://localhost:7984/_utils/

5.3.4.1 피어 노드 3개의 the world state에서 마지막 세가지 항목 비교 (그림참조 : One channel with private data collection defined for org1 and org2.)
- channel-all_mycc : 개인 데이터 수집을 사용하지 않는 상태의 public data가 모두 수집되는 곳이며, 모든 조직에 같은 데이터를 저장함.
- channel-all_mycc$$hcollection$private : the hash of the private data가 유지되며, 채널안의 모든 조직에서 볼수 있음.
- channel-all_mycc$$pcollection$private : keeps the actual private data, 오직 org1 and org2에서만 볼수 있음.

6. 참고사항 (그림참조 : Private data passed in chaincode invoke is still visible in the block.)
- Sending Private Data in Proposal 경우 createRecord Proposal의 인수로 the private data인 “passport” 정보가 트랜젝션 안에 존재하여 블록에 포함되어 생성됨 (Client -> Orderer).
- org3의 world state에서는 볼수 없으나, this transaction in the blockchain에서는 볼수 있음.
- Invoking a chaincode function 안에 private data를 사용하지 말고, transient data를 사용하는게 좋은 방법입니다. (자세한 사항은 백서를 참조하세요)

7. Channel vs. Private Data
- Channel : infrastructure level
- Private Data : data level
- 여러 채널내의 다양한 그룹들에게 데이터 레벨에서의 Private를 제공하기에는 multiple collections을 이용하는게 Channel의 단점을 극복하여 서비스 제공하기 쉬울수 있습니다. (주관적인 관점입니다)
- 참고 프로젝트 : Singapore Monetary Authority of Singapore (MAS) found in their Project Ubin Phase 2

8. Clean Up
- 5 단계 clean up : ./network-down.sh

$ cd first-network

$ ./byfn.sh down

$ docker rm $(docker ps -aq)

$ docker rmi $(docker images dev-* -q)

$ docker network prune



