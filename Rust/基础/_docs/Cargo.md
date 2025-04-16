## 概述

cargo是一个强大的rust包管理器, 提供了从项目建立, 构建到测试, 运行直至部署的一系列工具. 同时与Rust语言及其编译器rustc紧密结合

## 创建新项目

cargo new 项目名称

## 运行/编译项目

cargo run
cargo build
默认是debug模式, 编译快, 但是运行效率低, 没有优化

## 检查项目

cargo check 判断能否通过编译

## 文件

cargo.toml: 项目数据描述文件, 存储了项目的所有元配置
- \[package\] 
	- name: 项目名
	- version: 版本
	- edition: Rust大版本
- \[dependencies\] 管理依赖 
- rand = "0.3" 
- hammer = { version = "0.5.0"} 
- color = { git = "https://github.com/bjz/color-rs" } 
- geometry = { path = "crates/geometry" }
cargo.lock: 根据toml文件生成的项目依赖详细清单, 一般不需要修改