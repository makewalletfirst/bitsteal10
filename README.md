# 원본 비트코인(매직 f9beb4d9), 풀노드로 동기화
./src/bitcoind -datadir=/home/makewalletfirst/bitcoin \
  -server=1 -rpcuser=user -rpcpassword=pass \
  -prune=0 -daemon
./src/bitcoin-cli -datadir=/home/makewalletfirst/bitcoin -rpcuser=user -rpcpassword=pass getblockcount
# → 1296 이상까지 기다림
# 종료
./src/bitcoin-cli -datadir=/home/makewalletfirst/bitcoin -rpcuser=user -rpcpassword=pass stop

새 매직(e2c3a9fc) 부트스트랩 파일 생성 (레코드 단위 재패킹)
mk_bootstrap.py
# mk_bootstrap.py
import glob, struct, os, sys
SRC_DIR = "/home/makewalletfirst/bitcoin/blocks"   # 원본 blk*.dat 위치
OUT     = "/home/makewalletfirst/bootstrap-e2c3a9fc.dat"
OLD = bytes.fromhex("f9beb4d9")
NEW = bytes.fromhex("e2c3a9fc")

files = sorted(glob.glob(os.path.join(SRC_DIR, "blk*.dat")))
assert files, "no blk files"
n=0
with open(OUT, "wb") as out:
    for p in files:
        d=open(p,"rb").read(); i=0; L=len(d)
        while True:
            j=d.find(OLD,i)
            if j<0 or j+8>L: break
            (blen,)=struct.unpack_from("<I",d,j+4)
            k=j+8+blen
            if k>L: break
            out.write(NEW); out.write(struct.pack("<I",blen)); out.write(d[j+8:k])
            n+=1; i=k
print("wrote", n, "blocks to", OUT)

python3 mk_bootstrap.py
ls -lh /home/makewalletfirst/bootstrap-e2c3a9fc.dat
head -c 4 /home/makewalletfirst/bootstrap-e2c3a9fc.dat | xxd   # e2 c3 a9 fc 확인







consensus/params.h
// struct Consensus::Params 내부
int nForkHeight = std::numeric_limits<int>::max();
uint256 powLimitPostFork;
bool fPowAllowMinDifficultyBlocksPostFork = false;
bool fPowNoRetargetingPostFork = false;


chainparams.cpp – 메인넷 파라미터 설정
// CMainParams() 생성자 내부 (pow/spacing 등 근처)
pchMessageStart[0]=0xe2; pchMessageStart[1]=0xc3; pchMessageStart[2]=0xa9; pchMessageStart[3]=0xfc;
nDefaultPort = 8333;

consensus.nForkHeight = 1297;
consensus.powLimitPostFork = uint256S("0000ffff00000000000000000000000000000000000000000000000000000000");
consensus.fPowAllowMinDifficultyBlocksPostFork = true;
consensus.fPowNoRetargetingPostFork = true;

// 개발 단계에서는 외부 피어 차단
vSeeds.clear();
vFixedSeeds.clear();



pow.cpp
1) GetNextWorkRequired(...) 함수 맨 앞
const int nHeightNext = pindexLast ? (pindexLast->nHeight + 1) : 0;
if (nHeightNext >= params.nForkHeight) {
    // 포크 이후 즉시 최저 난이도
    return UintToArith256(params.powLimitPostFork).GetCompact();
}

2) CheckProofOfWork(...) 시그니처와 한도
bool CheckProofOfWork(uint256 hash, unsigned int nBits, const Consensus::Params& params)
{
    // 프리/포스트 포크 중 더 느슨한 powLimit 허용
    const arith_uint256 pre  = UintToArith256(params.powLimit);
    const arith_uint256 post = UintToArith256(params.powLimitPostFork);
    const arith_uint256 bnLimit = (pre > post) ? pre : post;

    arith_uint256 bnTarget; bool fNeg=false, fOv=false;
    bnTarget.SetCompact(nBits, &fNeg, &fOv);
    if (fNeg || bnTarget==0 || fOv || bnTarget > bnLimit) return false;
    if (UintToArith256(hash) > bnTarget) return false;
    return true;
}
주의: 함수 시그니처는 반드시 pow.h 선언과 같아야 합니다:
bool CheckProofOfWork(uint256 hash, unsigned int nBits, const Consensus::Params&);



노드 A – 부트스트랩 로드(새 매직), 안정화 후 재기동
# A의 datadir 초기화(기존 인덱스 제거)
rm -rf /home/makewalletfirst/myfork/blocks /home/makewalletfirst/myfork/chainstate

# 부트스트랩 로드
./src/bitcoind -datadir=/home/makewalletfirst/myfork \
  -server=1 -rpcuser=user -rpcpassword=pass \
  -dnsseed=0 -fixedseeds=0 -listen=1 -prune=0 \
  -loadblock=/home/makewalletfirst/bootstrap-e2c3a9fc.dat \
  -daemon -debug=validation

# debug.log에서 "Loaded N blocks from external file", "Done loading" 확인 후 재기동
./src/bitcoin-cli -datadir=/home/makewalletfirst/myfork -rpcuser=user -rpcpassword=pass stop
./src/bitcoind -datadir=/home/makewalletfirst/myfork \
  -server=1 -rpcuser=user -rpcpassword=pass \
  -dnsseed=0 -fixedseeds=0 -listen=1 -prune=0 \
  -daemon -debug=net


5) 노드 B – 새 매직, A로만 접속
mkdir -p /home/makewalletfirst/myfork2
./src/bitcoind -datadir=/home/makewalletfirst/myfork2 \
  -server=1 -rpcuser=user -rpcpassword=pass \
  -dnsseed=0 -fixedseeds=0 \
  -connect=34.64.45.122:8333 \
  -daemon -debug=net

6) 포크 블록 채굴(노드 A) & B 동기화 확인
# A에서 주소 만들고
ADDR=$(./src/bitcoin-cli -datadir=/home/makewalletfirst/myfork -rpcuser=user -rpcpassword=pass getnewaddress)
# 바로 채굴 (포크 높이=1297부터 난이도 최저)
./src/bitcoin-cli -datadir=/home/makewalletfirst/myfork -rpcuser=user -rpcpassword=pass generatetoaddress 1 "$ADDR"
# B에서 블록 따라왔는지 확인
./src/bitcoin-cli -datadir=/home/makewalletfirst/myfork2 -rpcuser=user -rpcpassword=pass getblockcount




  
   
