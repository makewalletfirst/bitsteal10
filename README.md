일단 끊어서 포크한 기점은 86367임

original bitcoin
getblock 86366
~~~b7f1e5a8b
getblock 86367
~~~bf2156224
getblock 86368
~~~fd268d56f

myfork
getblock 86366
~~~b7f1e5a8b
getblock 86367
~~~bf2156224
getblock 86368
~~~e427a1a3e2

cpp 그록이 준거에서 pow 변경

먼저 원래 코드로 동기화 원하는 블록까지 땡기고
mkdir ~/myfork
cp -r ~/.bitcoin/blocks ~/myfork/blocks
cp -r ~/.bitcoin/chainstate ~/myfork/chainstate

실행
./src/bitcoind -datadir=/home/makewalletfirst/myfork -daemon

블록확인
./src/bitcoin-cli -datadir=/home/makewalletfirst/myfork getblockcount

피어확인(피어잡히지만 그이상의 동기화 안함)
./src/bitcoin-cli -datadir=/home/makewalletfirst/myfork getpeerinfo

지갑생성
./src/bitcoin-cli -datadir=/home/makewalletfirst/myfork createwallet "wallet"

주소1개생성
./src/bitcoin-cli -datadir=/home/makewalletfirst/myfork getnewaddress

채굴 1개
./src/bitcoin-cli -datadir=/home/makewalletfirst/myfork generatetoaddress 1 $(./src/bitcoin-cli -datadir=/home/makewalletfirst/myfork getnewaddress)






-datadir=/home/makewalletfirst/myfork
