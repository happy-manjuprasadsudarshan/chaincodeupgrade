# create shell script for deployment of chaincode
# Usage: ./deploychaincode.sh <chaincode_name> <chaincode_version> <chaincode_path> <channel_name> <org_name> <peer_name> <orderer_name> <orderer_domain> <orderer_port> <orderer_tls_cacerts> <peer_domain> <peer_port> <peer_tls_cacerts> <peer_tls_cert> <peer_tls_key> <peer_tls_client_cert> <peer_tls_client_key> <peer_tls_rootcert

#steps:
#1. Install chaincode on peer
#2. Instantiate chaincode on channel
#3. Query chaincode on channel
#4. Invoke chaincode on channel

#steps we are following in install chaincode:
#1. install chaincode from each peer0 cli
#2. kubectl exec -it<cli-peer0-org1> -n=sidbi -- bash
#3. cd/opt/gopath/src/github.com/chaincode/sidbi/packaging
#4. peer chaincode install sidbi-org1.tgz
#5. copy chaincode id and replace chaincode id for org1 in notes.txt file
#6. replace chaincode id in 9.cc-deploy/sidbi/org1-chaincode-deployment.yaml
#7. kubectl apply -f 9.cc-deploy/sidbi/org1-chaincode-deployment.yaml
#8. kubectl exec -it<cli-peer0-org2> -n=sidbi -- bash
#9. cd/opt/gopath/src/github.com/chaincode/sidbi/packaging
#10. peer chaincode install sidbi-org2.tgz
#11. copy chaincode id and replace chaincode id for org2 in notes.txt file
#12. replace chaincode id in 9.cc-deploy/sidbi/org2-chaincode-deployment.yaml
#13. kubectl apply -f 9.cc-deploy/sidbi/org2-chaincode-deployment.yaml
#14. kubectl exec -it<cli-peer0-org3> -n=sidbi -- bash
#15. cd/opt/gopath/src/github.com/chaincode/sidbi/packaging
#16. peer chaincode install sidbi-org3.tgz
#17. copy chaincode id and replace chaincode id for org3 in notes.txt file
#18. replace chaincode id in 9.cc-deploy/sidbi/org3-chaincode-deployment.yaml
#19. kubectl apply -f 9.cc-deploy/sidbi/org3-chaincode-deployment.yaml

#create a shell script for upgrade of chaincode
#upgradechaincode.sh
#!/bin/bash

CHAINCODE_NAME=$1
CHAINCODE_VERSION=$2
CHAINCODE_PATH=$3
CHANNEL_NAME=$4
ORG_NAME=$5
PEER_NAME=$6
ORDERER_NAME=$7
ORDERER_DOMAIN=$8
ORDERER_PORT=$9
ORDERER_TLS_CACERTS=${10}
PEER_DOMAIN=${11}
PEER_PORT=${12}
PEER_TLS_CACERTS=${13}
PEER_TLS_CERT=${14}
PEER_TLS_KEY=${15}
PEER_TLS_CLIENT_CERT=${16}
PEER_TLS_CLIENT_KEY=${17}
PEER_TLS_ROOTCERT=${18}

# Function to install chaincode on a peer
install_chaincode() {
  local PEER_CLI=$1
  local CHAINCODE_PACKAGE=$2
  local ORG=$3

  kubectl exec -it $PEER_CLI -n=sidbi -- bash -c "
    cd /opt/gopath/src/github.com/chaincode/sidbi/packaging &&
    peer chaincode install $CHAINCODE_PACKAGE &&
    CHAINCODE_ID=\$(peer chaincode queryinstalled | grep $CHAINCODE_NAME | awk '{print \$3}') &&
    echo \$CHAINCODE_ID > /tmp/chaincode_id.txt &&
    sed -i 's/CHAINCODE_ID/'\$CHAINCODE_ID'/g' 9.cc-deploy/sidbi/$ORG-chaincode-deployment.yaml &&
    kubectl apply -f 9.cc-deploy/sidbi/$ORG-chaincode-deployment.yaml
  "
}

# Install chaincode on peer0 of org1
install_chaincode "cli-peer0-org1" "sidbi-org1.tgz" "org1"

# Install chaincode on peer0 of org2
install_chaincode "cli-peer0-org2" "sidbi-org2.tgz" "org2"

# Install chaincode on peer0 of org3
install_chaincode "cli-peer0-org3" "sidbi-org3.tgz" "org3"

# Approve chaincode definition for org1
kubectl exec -it cli-peer0-org1 -n=sidbi -- bash -c "
  peer chaincode approveformyorg -o $ORDERER_NAME.$ORDERER_DOMAIN:$ORDERER_PORT --tls --cafile $ORDERER_TLS_CACERTS --channelID $CHANNEL_NAME --name $CHAINCODE_NAME --version $CHAINCODE_VERSION --init-required --package-id \$(cat /tmp/chaincode_id.txt) --sequence 1 --waitForEvent
"

# Approve chaincode definition for org2
kubectl exec -it cli-peer0-org2 -n=sidbi -- bash -c "
  peer chaincode approveformyorg -o $ORDERER_NAME.$ORDERER_DOMAIN:$ORDERER_PORT --tls --cafile $ORDERER_TLS_CACERTS --channelID $CHANNEL_NAME --name $CHAINCODE_NAME --version $CHAINCODE_VERSION --init-required --package-id \$(cat /tmp/chaincode_id.txt) --sequence 1 --waitForEvent
"

# Approve chaincode definition for org3
kubectl exec -it cli-peer0-org3 -n=sidbi -- bash -c "
  peer chaincode approveformyorg -o $ORDERER_NAME.$ORDERER_DOMAIN:$ORDERER_PORT --tls --cafile $ORDERER_TLS_CACERTS --channelID $CHANNEL_NAME --name $CHAINCODE_NAME --version $CHAINCODE_VERSION --init-required --package-id \$(cat /tmp/chaincode_id.txt) --sequence 1 --waitForEvent
"

# check commit readiness
kubectl exec -it cli-peer0-org1 -n=sidbi -- bash -c "
  peer lifecycle chaincode checkcommitreadiness --channelID $CHANNEL_NAME --name $CHAINCODE_NAME --version $CHAINCODE_VERSION --sequence 1 --output json --init-required--sequence -o-orderer:7050 --tls--cafile $ORDERER_CA
"
#commit chaincode definition
#note: for every new upgrade of chaincode, we need to increment the version and sequence number
#our steps are:
1. kubectl exec -it <cli-peer0-org1> -n=sidbi -- bash
2. peer lifecycle chaincode commit -o orderer:7050 --channelID mychannel --name sidbi --version 2.0 --sequence 2 --init-required --tls --cafile $ORDERER_CA-- peerAddresses peeer0-org1:7051--tlsRootCertFiles /organizations/peerOrganizations/org1.example.com/peers/peer0-org1/tls/ca.crt--peerAddresses peer0-org2:7051 --tlsRootCertFiles /organizations/peerOrganizations/org2.example.com/peers/peer0-org2/tls/ca.crt--peerAddresses peer0-org3:7051 --tlsRootCertFiles /organizations/peerOrganizations/org3.example.com/peers/peer0-org3/tls/ca.crt

#commit chaincode definition
kubectl exec -it cli-peer0-org1 -n=sidbi -- bash -c "
  peer lifecycle chaincode commit -o $ORDERER_NAME.$ORDERER_DOMAIN:$ORDERER_PORT --channelID $CHANNEL_NAME --name $CHAINCODE_NAME --version $CHAINCODE_VERSION --sequence 1 --init-required --tls --cafile $ORDERER_TLS_CACERTS --peerAddresses $PEER_NAME.$PEER_DOMAIN:$PEER_PORT --tlsRootCertFiles $PEER_TLS_ROOTCERT
"

#invoke chaincode
kubectl exec -it cli-peer0-org1 -n=sidbi -- bash -c "
  peer chaincode invoke -o $ORDERER_NAME.$ORDERER_DOMAIN:$ORDERER_PORT --tls --cafile $ORDERER_TLS_CACERTS -C $CHANNEL_NAME -n $CHAINCODE_NAME --peerAddresses $PEER_NAME.$PEER_DOMAIN:$PEER_PORT --tlsRootCertFiles $PEER_TLS_ROOTCERT -c '{\"function\":\"initLedger\",\"Args\":[]}'

 peer chaincode query -C $CHANNEL_NAME -n $CHAINCODE_NAME -c '{\"function\":\"queryAllAssets\",\"Args\":[]}'
