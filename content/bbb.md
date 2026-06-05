+++
title = "豆瓣反爬机制分析"
description = "豆瓣反爬机制分析"
date = 2026-06-06
[taxonomies]
tags = ["技术", "markdown"]
+++

## HTTP交互分析

```bash
# @name first_request
# 显式禁用自动跟随重定向，这样就能在响应头中看到 302 和 Location 了
# @no-redirect
GET https://movie.douban.com/subject/36467231/ HTTP/1.1
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:150.0) Gecko/20100101 Firefox/150.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.9


###


# @name get_challenge_page
# 复制第一步返回的 Location 完整 URL 到这里
GET https://sec.douban.com/c?r=https%3A%2F%2Fmovie.douban.com%2Fsubject%2F36467231%2F&_s=2dc0d55be8e73ca434e59a345b1c942814d2c8e65d07b2c70025abde43f11a0a&a=1 HTTP/1.1
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:150.0) Gecko/20100101 Firefox/150.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.9


###


# @name submit_pow_answer
# 显式阻止自动跳转，以便我们在响应头里截获高价值的 dbsawcv1 Cookie
# @no-redirect
POST https://sec.douban.com/c HTTP/1.1
Host: sec.douban.com
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:150.0) Gecko/20100101 Firefox/150.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.9
Content-Type: application/x-www-form-urlencoded
# 伪装来源指纹，告诉网关你是从刚刚那个 get_challenge_page 页面填完答案提交的
Referer: https://sec.douban.com/c?r=https%3A%2F%2Fmovie.douban.com%2Fsubject%2F36467231%2F&_s=2dc0d55be8e73ca434e59a345b1c942814d2c8e65d07b2c70025abde43f11a0a&a=1

tok=1780539446@5fb8bf27582d1ef64ee6b768c022d278d54c387f4bcc5c818456ef44d7ccaad59f672a4e31ad065578b4ec7b1a71bd80ebbada02185c83ea66b58009eb2f0ed8@aHR0cHM6Ly9tb3ZpZS5kb3ViYW4uY29tL3N1YmplY3QvMzY0NjcyMzEv@838b60820b69e6e1eaf61b824f3e78a47ab59ffc8db6dc814d6568473e452d2a&cha=d1609ca44747933dfe98254c76b83bdc9b9cb20ebb220b7089150f14cd8d9773218c1842b41634f1d855a27f030d1a4ab5c762cc9c2e4ea2ba2b9c25c14844f7&sol=21445&red=https%3A%2F%2Fmovie.douban.com%2Fsubject%2F36467231%2F


###

# @name fetch_movie_detail
# 这一步不需要 @no-redirect，我们直接要看 200 OK 里的真实电影网页
GET https://movie.douban.com/subject/36467231/ HTTP/1.1
Host: movie.douban.com
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:150.0) Gecko/20100101 Firefox/150.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: zh-CN,zh;q=0.9
# 将刚刚拦截到的通行证附加在请求头中
Cookie: dbsawcv1=MTc4MDUzOTU2NkA4MzhiNjA4MjBiNjllNmUxZWFmNjFiODI0ZjNlNzhhNDdhYjU5ZmZjOGRiNmRjODE0ZDY1Njg0NzNlNDUyZDJhQGQ5Njg3Mzk2Y2EwNjVhNzZAZDEzMjIxOWQzMjQ4

```

## 获取POW脚本

```javascript

<script>

function sha512(string) {
    return new Promise((resolve, reject) => {
        let buffer = (new TextEncoder).encode(string);

        crypto.subtle.digest('SHA-512', buffer.buffer).then(result => {
            resolve(Array.from(new Uint8Array(result)).map(
                c => c.toString(16).padStart(2, '0')
            ).join(''));
        }, reject);
    });
}

async function process(data, difficulty = 4) {
    let hash;
    let nonce = 0;
    const targetSubStr = Array(difficulty+1).join('0');

    do {
        nonce += 1;
        hash = await sha512(data + nonce);
    } while(hash.substr(0, difficulty) !== targetSubStr);
    return nonce;
}

async function run(e) {
    e.preventDefault();
    const tok = document.querySelector("#tok").value;
    const cha = document.querySelector("#cha").value;
    let f = document.querySelector("#sol");
    let s = await process(cha);
    f.value = s;
    e.target.submit();
}

window.addEventListener("load", function chk(e) {
    const sub = document.querySelector("#sec");
    sub.addEventListener("submit", run);
    setTimeout(() => { sub.requestSubmit(); }, 500);
})
</script>



```

## RUST爬虫验证

```rust

use reqwest::header::{HeaderMap, HeaderValue, USER_AGENT, ACCEPT, ACCEPT_LANGUAGE};
use regex::Regex;
use sha2::{Sha512, Digest};
use std::sync::Arc;
use std::time::Instant;

// 模拟真实浏览器的 User-Agent
const USER_AGENT_STR: &str = "Mozilla/5.0 (X11; Linux x86_64; rv:150.0) Gecko/20100101 Firefox/150.0";

/// 本地 PoW 算力碰撞函数
/// 寻找一个 sol (nonce)，使得 sha512(cha + sol) 的前 4 位字符为 '0'
fn solve_pow(cha: &str) -> u64 {
    let start_time = Instant::now();
    let mut sol: u64 = 0;

    println!("[*] 开始计算 PoW 验证答案...");

    loop {
        // 1. 拼接挑战值 cha 与当前尝试的 sol
        let input = format!("{}{}", cha, sol);

        println!("{input}");

        // 2. 创建 SHA-512 哈希计算实例并注入数据
        let mut hasher = Sha512::new();
        hasher.update(input.as_bytes());
        let result = hasher.finalize();

        // 3. 将二进制哈希结果转换为小写十六进制字符串
        let hex_res = format!("{:x}", result);

        println!("{hex_res}");
        // 4. 判断前 4 位字符是否全为 '0'（满足难度要求）
        if hex_res.starts_with("0000") {
            println!("[+] 碰撞成功！用时: {:?}, 找到解 sol = {}", start_time.elapsed(), sol);
            return sol;
        }

        // 5. 若不匹配，则 sol 递增，进入下一次循环盲猜
        sol += 1;
    }
}

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // 目标电影页面路径
    let target_url = "https://movie.douban.com/subject/36467231/";

    // 初始化带有自动 Cookie 容器（Jar）的 HTTP 客户端
    let jar = Arc::new(reqwest::cookie::Jar::default());
    let client = reqwest::Client::builder()
        .cookie_provider(jar.clone())
        .build()?;

    // 配置伪装请求头
    let mut headers = HeaderMap::new();
    headers.insert(USER_AGENT, HeaderValue::from_static(USER_AGENT_STR));
    headers.insert(ACCEPT, HeaderValue::from_static("text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8"));
    headers.insert(ACCEPT_LANGUAGE, HeaderValue::from_static("zh-CN,zh;q=0.9"));

    // -------------------------------------------------------------
    // 步骤一：发起初始请求，触发并跟随 302 重定向到验证页面
    // -------------------------------------------------------------
    println!("[1] 正在请求目标页面并等待重定向至验证网关...");
    let response = client.get(target_url)
        .headers(headers.clone())
        .send()
        .await?;

    let body_text = response.text().await?;

    // -------------------------------------------------------------
    // 步骤二：使用正则表达式解析 HTML，提取验证所需的关键参数
    // -------------------------------------------------------------
    let cha_regex = Regex::new(r#"name="cha"\s+value="([^"]+)""#)?;
    let tok_regex = Regex::new(r#"name="tok"\s+value="([^"]+)""#)?;

    let cha = match cha_regex.captures(&body_text) {
        Some(cap) => cap.get(1).unwrap().as_str().to_string(),
        None => {
            println!("[-] 未在页面中找到 cha 参数，可能已被放行，无需验证");
            return Ok(());
        }
    };

    let tok = match tok_regex.captures(&body_text) {
        Some(cap) => cap.get(1).unwrap().as_str().to_string(),
        None => {
            println!("[-] 未在页面中找到 tok 参数");
            return Ok(());
        }
    };

    println!("[+] 成功提取网关参数:\n    cha: {}\n    tok: {}", cha, tok);

    // -------------------------------------------------------------
    // 步骤三：调用本地高性能 CPU 循环，爆破 PoW 谜题
    // -------------------------------------------------------------
    let sol = solve_pow(&cha);

    // -------------------------------------------------------------
    // 步骤四：伪造浏览器表单提交，把答案 POST 给安全验证网关
    // -------------------------------------------------------------
    let verify_url = "https://sec.douban.com/c";
    println!("[3] 正在向验证网关提交算力答案...");

    let params = [
        ("req_path", "/subject/36467231/"),
        ("cha", &cha),
        ("tok", &tok),
        ("sol", &sol.to_string()),
    ];

    // 发送 POST 表单。网关校验成功后会返回 302 并下发 set-cookie: dbsawcv1
    // reqwest 客户端会自动把此 Cookie 存入我们先前初始化的 jar 中
    let verify_res = client.post(verify_url)
        .headers(headers.clone())
        .form(&params)
        .send()
        .await?;

    println!("[+] 验证提交完成，网关响应状态码: {}", verify_res.status());

    // -------------------------------------------------------------
    // 步骤五：Cookie 容器已持有合法凭证，二次请求目标页面获取数据
    // -------------------------------------------------------------
    println!("[4] 正在携带安全 Cookie 重新请求电影详情页...");
    let final_res = client.get(target_url)
        .headers(headers)
        .send()
        .await?;

    let final_html = final_res.text().await?;


    // println!("{final_html}");
    // 验证成果：提取并打印网页标题
    if final_html.contains("<title>") {
        let title_regex = Regex::new(r"<title>(.*?)</title>")?;
        // println!("ok")
        if let Some(cap) = title_regex.captures(&final_html) {
            println!("\n[🎉 抓取成功!] 页面标题为: {}", cap.get(1).unwrap().as_str().trim());
        }
    } else {
        println!("[-] 抓取失败，未能成功绕过验证。");
    }

    Ok(())
}


```
