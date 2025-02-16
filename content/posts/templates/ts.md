---
<%*
// 1. 파일 이름 변경 로직
// 현재 파일의 경로를 가져옵니다.
const filePath = tp.file.path(true);

// 경로를 '/'로 분리합니다.
const pathParts = filePath.split('/');

// 폴더 이름을 가져옵니다.
const folderName = pathParts[pathParts.length - 2];

// 파일 제목을 설정합니다.
await tp.file.rename('index');
-%>
title: <% folderName %>
date: <% tp.date.now("YYYY-MM-DD") %>
tags: 
categories: 
summary:
---

## 문제 정의

## 문제 분석

## 문제 해결 과정

## 개선 결과

## 결론