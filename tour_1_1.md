## 本地安装solana环境

### 当前环境
windows11

### 可用的环境安装&运行方式
1. windows-主机环境安装&运行 
2. windows-WSL环境安装&运行
3. windows-docker环境安装&运行

### windows-主机环境安装&运行

前面安装都比较顺利，但是运行不行编译不过去
原因如下
Solana Programs（即 on-chain 程序）是用 Rust 编写，但不是编译成普通 x86 二进制，而是编译成 BPF bytecode。
BPF 工具链（cargo-build-bpf、rustc-bpf 等）是 Linux / macOS 原生的，依赖 LLVM 和一些 POSIX 工具。
Windows 没有完整的 POSIX 环境，BPF 工具链无法正常运行。

### windows-WSL环境安装&运行
前面一如既往的顺利，在领空投和获取余额上卡住了
```
ichael@localhost:~$ solana balance
Error: error sending request for url (https://api.devnet.solana.com/)
```

因为健康上网的原因，调试了半天用windows的代理不行，所以放弃


### windows-docker环境安装&运行
docker下是成功了的
docker下安装首先要找镜像，这里推荐两个镜像
1. `pylejeune/solana-dev:anchor-0.31.1`
2. `solanafoundation/anchor`

镜像拉取成功后启动容器
` docker run -it --rm -v ${PWD}:/workspace -w /workspace pylejeune/solana-dev:anchor-0.31.1 bash
`

在`anchor build`的时候又出幺蛾子了

```
error: rustc 1.79.0-dev is not supported by the following package:
                 Note that this is the rustc version that ships with Solana tools and not your system's rustc version. Use `solana-install update` or head over to https://docs.solanalabs.com/cli/install to install a newer version.
  indexmap@2.12.0 requires rustc 1.82
Either upgrade rustc or select compatible dependency versions with
`cargo update <name>@<current-ver> --precise <compatible-ver>`
where `<compatible-ver>` is the latest version supporting rustc 1.79.0-dev
```

查看rust版本确实1.85.0，只得继续问ChatGPT，ChatGPT有时候回答的不准。
尝试了多种办法后，直接升级solana的版本即可

`sh -c "$(curl -sSfL https://release.anza.xyz/stable/install)"`

### 写一个投票系统
1. 第一步：初始化项目
`anchor init solana_vote2`

2. 第二步：编写lib
文件路径：
`programs/solana_vote2/src/lib.rs`

文件内容：
```
use anchor_lang::prelude::*;


declare_id!("12222");

#[program]
pub mod vote_program {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>, candidates: Vec<String>) -> Result<()> {
        let vote_account = &mut ctx.accounts.vote_account;
        vote_account.candidates = candidates;
        vote_account.votes = vec![0; vote_account.candidates.len()];
        Ok(())
    }

    pub fn vote(ctx: Context<Vote>, candidate_index: u8) -> Result<()> {
        let vote_account = &mut ctx.accounts.vote_account;

        if candidate_index as usize >= vote_account.votes.len() {
            return Err(ErrorCode::InvalidCandidate.into());
        }

        vote_account.votes[candidate_index as usize] += 1;
        Ok(())
    }
}

#[account]
pub struct VoteAccount {
    pub candidates: Vec<String>,
    pub votes: Vec<u32>,
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(init, payer = user, space = 9000)] // 注意 space 大小，可根据候选人数量调整
    pub vote_account: Account<'info, VoteAccount>,
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Vote<'info> {
    #[account(mut)]
    pub vote_account: Account<'info, VoteAccount>,
}

#[error_code]
pub enum ErrorCode {
    #[msg("Candidate index out of bounds")]
    InvalidCandidate,
}
```

1. 第三步：编写TEST
文件路径：
`tests/solana_vote2.ts`

文件内容
```
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { VoteProgram } from "../target/types/vote_program";
import { assert } from "chai";

describe("vote_program", () => {
  const provider = anchor.AnchorProvider.env();
  anchor.setProvider(provider);

  const program = anchor.workspace.VoteProgram as Program<VoteProgram>;
  const voteAccount = anchor.web3.Keypair.generate();

  // 候选人列表
  const candidates = ["Alice", "Bob", "Charlie"];

  // 初始化投票
  it("Initialize voting", async () => {
    await program.methods.initialize(candidates)
      .accounts({
        voteAccount: voteAccount.publicKey,
        user: provider.wallet.publicKey,
        systemProgram: anchor.web3.SystemProgram.programId,
      })
      .signers([voteAccount])
      .rpc();

    const account = await program.account.voteAccount.fetch(voteAccount.publicKey);
    console.log("Candidates:", account.candidates);
    console.log("Votes:", account.votes);

    // 断言初始票数为 0
    assert.deepEqual(account.votes, [0, 0, 0]);
  });

  // 投票函数
  async function vote(candidateIndex: number) {
    await program.methods.vote(candidateIndex)
      .accounts({ voteAccount: voteAccount.publicKey })
      .rpc();

    const account = await program.account.voteAccount.fetch(voteAccount.publicKey);
    console.log(`Vote for ${account.candidates[candidateIndex]}:`, account.votes);
    return account;
  }

  // 投票示例
  it("Vote for Alice", async () => {
    const account = await vote(0);
    assert.equal(account.votes[0], 1);
  });

  it("Vote for Bob twice", async () => {
    await vote(1);
    const account = await vote(1);
    assert.equal(account.votes[1], 2);
  });

  it("Vote for Charlie", async () => {
    const account = await vote(2);
    const accountAfter = await program.account.voteAccount.fetch(voteAccount.publicKey);
    assert.deepEqual(accountAfter.votes, [1, 2, 1]);
  });
});
```
4. 第四步：编译&测试
`anchor build`
`anchor test` 


### 运行效果

```
Found a 'test' script in the Anchor.toml. Running it as a test suite!

Running test suite: "/workspace/solana_vote2/Anchor.toml"

yarn run v1.22.22
$ /workspace/solana_vote2/node_modules/.bin/ts-mocha -p ./tsconfig.json -t 1000000 'tests/**/*.ts'
  vote_program
Candidates: [ 'Alice', 'Bob', 'Charlie' ]
Votes: [ 0, 0, 0 ]
    ✔ Initialize voting (197ms)
Vote for Alice: [ 1, 0, 0 ]
    ✔ Vote for Alice (413ms)
Vote for Bob: [ 1, 1, 0 ]
Vote for Bob: [ 1, 2, 0 ]
    ✔ Vote for Bob twice (874ms)
Vote for Charlie: [ 1, 2, 1 ]
    ✔ Vote for Charlie (465ms)
  4 passing (2s)
Done in 21.75s.
```
