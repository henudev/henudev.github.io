---
layout: post                          # 表明是博文  
title: "Spring Security执行过程浅析"           # 博文的标题  
date: 2022-05-12                 # 博文的发表日期，此日期决定主页上博文的先后顺序  
author: "秦"                       # 博文的作者  
catalog: True                         # 开启catalog，将在博文侧边展示博文的结构 
tags:
    - Spring
    - 分析
---  
# Spring Security执行过程浅析

[TOC]

## 基础概念

### 无状态：

- 服务端不保存任何客户端请求者信息
- 客户端的每次请求必须具备自描述信息，通过这些信息识别客户端身份

## 核心组件

### SecurityContextPersistenceFilter

主要工作是从session中获取SecurityContext，然后放到上下文中,之后的filter大多依赖这个来获取登录信息，其主要是通过HttpSessionSecurityContextRepository来存取的。

### HttpSessionSecurityContextRepository

## 重要流程

### JWT机制

当一个前端请求到达后端，被security框架拦截后，首先进入SecurityContextPersistenceFilter，然后通过HttpSessionSecurityContextRepository来获取SpringContext上下文，如果没有使用

### Session机制

当一个前端请求到达后端，被security框架拦截后，首先进入SecurityContextPersistenceFilter，然后通过HttpSessionSecurityContextRepository来获取SpringContext上下文，

*SecurityContextPersistenceFilter源码如下：*

```java
/*
 * Copyright 2002-2016 the original author or authors.
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package org.springframework.security.web.context;

import java.io.IOException;

import javax.servlet.FilterChain;
import javax.servlet.ServletException;
import javax.servlet.ServletRequest;
import javax.servlet.ServletResponse;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;
import javax.servlet.http.HttpSession;

import org.springframework.security.core.context.SecurityContext;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.web.filter.GenericFilterBean;

/**
 * Populates the {@link SecurityContextHolder} with information obtained from the
 * configured {@link SecurityContextRepository} prior to the request and stores it back in
 * the repository once the request has completed and clearing the context holder. By
 * default it uses an {@link HttpSessionSecurityContextRepository}. See this class for
 * information <tt>HttpSession</tt> related configuration options.
 * <p>
 * This filter will only execute once per request, to resolve servlet container
 * (specifically Weblogic) incompatibilities.
 * <p>
 * This filter MUST be executed BEFORE any authentication processing mechanisms.
 * Authentication processing mechanisms (e.g. BASIC, CAS processing filters etc) expect
 * the <code>SecurityContextHolder</code> to contain a valid <code>SecurityContext</code>
 * by the time they execute.
 * <p>
 * This is essentially a refactoring of the old
 * <tt>HttpSessionContextIntegrationFilter</tt> to delegate the storage issues to a
 * separate strategy, allowing for more customization in the way the security context is
 * maintained between requests.
 * <p>
 * The <tt>forceEagerSessionCreation</tt> property can be used to ensure that a session is
 * always available before the filter chain executes (the default is <code>false</code>,
 * as this is resource intensive and not recommended).
 *
 * @author Luke Taylor
 * @since 3.0
 */
public class SecurityContextPersistenceFilter extends GenericFilterBean {

	static final String FILTER_APPLIED = "__spring_security_scpf_applied";

	private SecurityContextRepository repo;

	private boolean forceEagerSessionCreation = false;

	public SecurityContextPersistenceFilter() {
		this(new HttpSessionSecurityContextRepository());
	}

	public SecurityContextPersistenceFilter(SecurityContextRepository repo) {
		this.repo = repo;
	}

	public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain)
			throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) req;
		HttpServletResponse response = (HttpServletResponse) res;

		if (request.getAttribute(FILTER_APPLIED) != null) {
			// ensure that filter is only applied once per request
			chain.doFilter(request, response);
			return;
		}

		final boolean debug = logger.isDebugEnabled();

		request.setAttribute(FILTER_APPLIED, Boolean.TRUE);

		if (forceEagerSessionCreation) {
			HttpSession session = request.getSession();

			if (debug && session.isNew()) {
				logger.debug("Eagerly created session: " + session.getId());
			}
		}

		HttpRequestResponseHolder holder = new HttpRequestResponseHolder(request,
				response);
		SecurityContext contextBeforeChainExecution = repo.loadContext(holder);

		try {
			SecurityContextHolder.setContext(contextBeforeChainExecution);

			chain.doFilter(holder.getRequest(), holder.getResponse());

		}
		finally {
			SecurityContext contextAfterChainExecution = SecurityContextHolder
					.getContext();
			// Crucial removal of SecurityContextHolder contents - do this before anything
			// else.
			SecurityContextHolder.clearContext();
			repo.saveContext(contextAfterChainExecution, holder.getRequest(),
					holder.getResponse());
			request.removeAttribute(FILTER_APPLIED);

			if (debug) {
				logger.debug("SecurityContextHolder now cleared, as request processing completed");
			}
		}
	}

	public void setForceEagerSessionCreation(boolean forceEagerSessionCreation) {
		this.forceEagerSessionCreation = forceEagerSessionCreation;
	}
}
```

### JwtLoginFilter 

1. 自定义 JwtLoginFilter 继承自 AbstractAuthenticationProcessingFilter，并实现其中的三个默认方法。
2. attemptAuthentication 方法中，我们从登录参数中提取出用户名密码，然后调用 AuthenticationManager.authenticate() 方法去进行自动校验。
3. 第二步如果校验成功，就会来到 successfulAuthentication 回调中，在 successfulAuthentication 方法中，将用户角色遍历然后用一个 `,` 连接起来，然后再利用 Jwts 去生成 token，按照代码的顺序，生成过程一共配置了四个参数，分别是用户角色、主题、过期时间以及加密算法和密钥，然后将生成的 token 写出到客户端。
4. 第二步如果校验失败就会来到 unsuccessfulAuthentication 方法中，在这个方法中返回一个错误提示给客户端即可。

### JwtFilter 

1. 首先从请求头中提取出 authorization 字段，这个字段对应的 value 就是用户的 token。
2. 将提取出来的 token 字符串转换为一个 Claims 对象，再从 Claims 对象中提取出当前用户名和用户角色，创建一个 UsernamePasswordAuthenticationToken 放到当前的 Context 中，然后执行过滤链使请求继续执行下去。

### filter链执行顺序

![执行顺序](https://github.com/henudev/henudev.github.io/blob/master/_posts/SpringSecurity.assets/image-20220422101832422.png?raw=true)

## 疑惑解答

### 为什么使用JWT还要使用

### 禁用session的时候，是么时候使用SessionCreationPolicy.STATELESS

1. 应用全生命周期不创建session
2. 应用全生命周期不使用session
