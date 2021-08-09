---
title:       "Golang 单元测试"
subtitle:    ""
description: ""
date:        2021-08-04
author:      "robinfan"
image:       ""
tags:        ["Golang", "单元测试"]
categories:  ["技术随笔" ]
---

内容大纲
----

*   单元测试的重要性
    
*   Go 单元测试基础知识
    
*   表格测试和 HTTP 测试
    
*   其它测试框架
    
*   其它 Mock 库
    
*   与 Docker 集成
    

单测的重要性
------

*   能够尽早的发现 bug
    
*   方便 debugging
    
*   方便代码重构
    
*   提升代码质量
    
*   使整个过程敏捷
    

讲这部分的目的是为了让团队达成共识 --- **单测很重要，我们必须要做好单元测试**。



Go 单测基础知识
---------

### 基本规则

*   通常我们的单元测试代码都放在以 **_test.go** 结尾的文件中，该文件一般和目标代码放在同一个 package 中。
    
*   测试的方法以为 **Test** 开头，并且拥有唯一一个 ***Testing.T** 的参数**。**
    

### `go test` 使用

*   `go test` 测试当前包
    
*   `go test some/pkg`  测试一个特定的包
    
*   `go test some/pkg/...` 递归测试一个特定包下面的所有包
    
*   `go test -v some/pkg -run ^TestSum$` 测试特定包下面的特定方法
    
*   `go test -cover` 查看单测覆盖率
    
*   `go test -count=1` 忽略缓存运行单测，注意如果以递归方式（./...）测试的时候，默认会使用 cache
    

### 表格测试

*   使用匿名结构体批量构建自己的测试 case
    
*   采用子测试的方式让测试输出更友好
    

```go
func TestIsIPV4WithTable(t *testing.T) {
	testCases := []struct {
		IP    string
		valid bool
	}{
		{"", false},
		{"192.168.0", false},
		{"192.168.x.1", false},
		{"192.168.0.1.1", false},
		{"127.0.0.1", true},
		{"192.168.0.1", true},
		{"255.255.255.255", true},
		{"120.52.148.118", true},
	}

	for _, tc := range testCases {
		t.Run(tc.IP, func(t *testing.T) {
			if IsIPV4(tc.IP) != tc.valid {
				t.Errorf("IsIPV4(%s) should be %v", tc.IP, tc.valid)
			}
		})
	}
}
```

复制代码

### HTTP 测试

*   使用 `httptest.NewRecorder` 来测试 HTTP Handler 而不需要真正进行 HTTP 监听
    
*   可以使用 `errorReader` 来提高测试覆盖率
    

```go
type errorReader struct{}

func (errorReader) Read(p []byte) (n int, err error) {
	return 0, errors.New("mock body error")
}

func TestLoginHandler(t *testing.T) {

	testCases := []struct {
		Name string
		Code int
		Body interface{}
	}{
		{"ok", 200, `{"code":"a@example.com", "password":"password"}`},
		{"read body error", 500, new(errorReader)},
		{"invalid format", 400, `{"code":1, "password":"password"}`},
		{"invalid code", 400, `{"code":"a@example.com1", "password":"password"}`},
		{"invalid password", 400, `{"code":"a@example.com", "password":"password1"}`},
	}

	for _, tc := range testCases {
		t.Run(tc.Name, func(t *testing.T) {

			var body io.Reader
			if stringBody, ok := tc.Body.(string); ok {
				body = strings.NewReader(stringBody)
			} else {
				body = tc.Body.(io.Reader)
			}

			req := httptest.NewRequest("POST", "http://example.com/foo", body)
			w := httptest.NewRecorder()

			LoginHandler(w, req)

			resp := w.Result()
			if resp.StatusCode != tc.Code {
				t.Errorf("response code is invalid, expect=%d but got=%d",
					tc.Code, resp.StatusCode)
			}
		})
	}
}
```

复制代码

**Go 单测基础知识**这部分的内容主要是向大家讲解 Go 官方单测库 [testing](https://golang.org/pkg/testing) 以及命令行 [go test](https://golang.org/pkg/cmd/go/internal/test) 的使用。

可以看到官方自带的库已经足够好用， 不仅带有 subtest 还有 httptest 的相关内容，对于中小型的项目而言使用官方的 testing 库足够。

其它测试框架
------

虽然官方 `testing` 库足够优秀，但在一些较大项目上，它还是有很多需要完善的地方，如：

*   断言不够友好，需要通过大量 if
    
*   持续集成不够，每次都要手动跑测试
    
*   BDD 无支持
    
*   测试 case 的文档自动化不够
    

所以我这里介绍了三种测试框架，针对以上几点都有一定的改进。

### Testify

*   和 `go test` 无缝集成，直接使用该命令运行
    
*   支持断言，写法更简便
    
*   支持 mocking
    
*   [https://github.com/stretchr/testify](https://github.com/stretchr/testify) （10k+ 关注）
    

```go
func TestIsIPV4WithTestify(t *testing.T) {
	assertion := assert.New(t)

	assertion.False(IsIPV4(""))
	assertion.False(IsIPV4("192.168.0"))
	assertion.False(IsIPV4("192.168.x.1"))
	assertion.False(IsIPV4("192.168.0.1.1"))
	assertion.True(IsIPV4("127.0.0.1"))
	assertion.True(IsIPV4("192.168.0.1"))
	assertion.True(IsIPV4("255.255.255.255"))
	assertion.True(IsIPV4("120.52.148.118"))
}
```

复制代码

### GoConvey

*   [https://github.com/smartystreets/goconvey](https://github.com/smartystreets/goconvey) （5k+ 关注）
    
*   支持 BDD
    
*   能够使用 `go test` 来运行测试
    
*   能够通过浏览器查看测试结果
    
*   自动加载更新
    

```go
func TestIsIPV4WithGoconvey(t *testing.T) {
	Convey("ip.IsIPV4()", t, func() {
		Convey("should be invalid", func() {
			Convey("empty string", func() {
				So(IsIPV4(""), ShouldEqual, false)
			})

			Convey("with less length", func() {
				So(IsIPV4("192.0.1"), ShouldEqual, false)
			})

			Convey("with more length", func() {
				So(IsIPV4("192.168.1.0.1"), ShouldEqual, false)
			})

			Convey("with invalid character", func() {
				So(IsIPV4("192.168.x.1"), ShouldEqual, false)
			})
		})

		Convey("should be valid", func() {
			Convey("loopback address", func() {
				So(IsIPV4("127.0.0.1"), ShouldEqual, true)
			})

			Convey("extranet address", func() {
				So(IsIPV4("120.52.148.118"), ShouldEqual, true)
			})
		})
	})
}
```



### GinkGo

*   [https://github.com/onsi/ginkgo](https://github.com/onsi/ginkgo) （4K+ 关注）
    
*   也是一个 BDD 测试框架
    
*   能够使用 `go test`
    
*   有自己的断言库 `Gomega`  
    
*   也支持自动加载更新
    

```go
var _ = Describe("Ip", func() {
	Describe("IsIPV4()", func() {
		// fore content level prepare
		BeforeEach(func() {
			// prepare data before every case
		})

		AfterEach(func() {
			// clear data after every case
		})

		Context("should be invalid", func() {
			It("empty string", func() {
				Expect(IsIPV4("")).To(Equal(false))
			})

			It("with less length", func() {
				Expect(IsIPV4("192.0.1")).To(Equal(false))
			})

			It("with more length", func() {
				Expect(IsIPV4("192.168.1.0.1")).To(Equal(false))
			})

			It("with invalid character", func() {
				Expect(IsIPV4("192.168.x.1")).To(Equal(false))
			})
		})

		Context("should be valid", func() {
			It("loopback address", func() {
				Expect(IsIPV4("127.0.0.1")).To(Equal(true))
			})

			It("extranet address", func() {
				Expect(IsIPV4("120.52.148.118")).To(Equal(true))
			})
		})
	})
})

func TestGinkgotesting(t *testing.T) {
	RegisterFailHandler(Fail)
	RunSpecs(t, "Ginkgotesting Suite")
}
```



其它 Mock 库
---------

到目前为止我们已经掌握了 Go 官方库 testing 和其它常见的测试框架的用法，能够方便我们编写和运行常规的单元测试。

但我们的系统往往比较复杂，依赖很多服务和基础组建，比如一个 Web 服务往往依赖 MySQL、Redis 等，这里主要讲解采用模拟（mock - 屏蔽掉这些服务的实际调用）的方式来测试我们的代码逻辑。

### GoMock

*   [https://github.com/golang/mock](https://github.com/golang/mock) （4k+ 关注）
    
*   golang 官方推出的 mock 库
    
*   针对接口进行 mock
    
*   支持 mock 和 stub
    
*   采用 `mockgen` 生成代码
    

```go
func TestPostIndexWithGoMock(t *testing.T) {
	ctrl := gomock.NewController(t)
	defer ctrl.Finish()

	Convey("PostController.Index", t, func() {
		Convey("should be 200", func() {
			posts := []*post.PostModel{
				{1, "title", "body"},
				{2, "title2", "body2"},
			}

			m := NewMockPostService(ctrl)
			m.
				EXPECT().
				List().
				Return(posts, nil)

			handler := post.PostController{
				PostService: m,
			}

			req := httptest.NewRequest("GET", "http://example.com/foo", nil)
			w := httptest.NewRecorder()

			handler.Index(w, req)

			So(w.Result().StatusCode, ShouldEqual, 200)
		})

		Convey("should be 500", func() {
			m := NewMockPostService(ctrl)
			m.
				EXPECT().
				List().
				Return(nil, errors.New("list post with error"))

			handler := post.PostController{
				PostService: m,
			}

			req := httptest.NewRequest("GET", "http://example.com/foo", nil)
			w := httptest.NewRecorder()
			handler.Index(w, req)
			So(w.Result().StatusCode, ShouldEqual, 500)
		})
	})
}
```



### HTTPMock

*   [https://github.com/jarcoal/httpmock](https://github.com/jarcoal/httpmock) （1K 关注）
    
*   针对 HTTP Request 来进行 mocking
    
*   能够自定义任意的 HTTP Response
    
*   截止 HTTP Request 直接返回自定义的 Response
    
*   给予正则匹配
    

```go
func TestPostClientFetch(t *testing.T) {
	httpmock.Activate()
	defer httpmock.DeactivateAndReset()

	postFetchURL := "https://api.mybiz.com/posts"

	client := &PostClient{
		Client: &http.Client{
			Transport: httpmock.DefaultTransport,
		},
	}

	Convey("PostClient.Fetch", t, func() {
		Convey("without error", func() {
			httpmock.RegisterResponder("GET", postFetchURL,
				httpmock.NewStringResponder(200, `[{"id": 1, "title": "title", "body": "body"}]`))

			items, err := client.Fetch(postFetchURL, 1)
			So(len(items), ShouldEqual, 1)
			So(err, ShouldEqual, nil)
		})

		Convey("with error", func() {
			Convey("response data invalid", func() {
				httpmock.RegisterResponder("GET", postFetchURL,
					httpmock.NewStringResponder(200, `[{"id": "213"}]`))

				items, err := client.Fetch(postFetchURL, 1)
				So(items, ShouldBeEmpty)
				So(err, ShouldNotBeNil)
			})

			Convey("without error", func() {
				httpmock.RegisterResponder("GET", postFetchURL,
					httpmock.NewStringResponder(500, `some error`))

				items, err := client.Fetch(postFetchURL, 1)
				So(items, ShouldBeEmpty)
				So(err.Error(), ShouldContainSubstring, "some error")
			})
		})
	})
}
```



### SQLMock

*   [https://github.com/DATA-DOG/go-sqlmock](https://github.com/DATA-DOG/go-sqlmock) （2k+ 关注）
    
*   针对 `database/sql` 的所有接口进行 mock
    
*   基于正则表达式进行匹配
    
*   支持 查询、更新、事务等 mock
    

```go
func TestPostDaoList(t *testing.T) {
	db, mock, err := sqlmock.New()
	if err != nil {
		t.Fatalf("an error '%s' was not expected when opening a stub database connection", err)
	}
	defer db.Close()

	Convey("PostDao.Fetch", t, func() {
		dao := post.NewPostDao(db)

		Convey("should be successful", func() {
			rows := sqlmock.NewRows([]string{"id", "title", "body"}).
				AddRow(1, "post 1", "hello").
				AddRow(2, "post 2", "world")
			mock.ExpectQuery("^SELECT (.+) FROM posts$").
				WithArgs().WillReturnRows(rows)

			items, err := dao.List()
			So(items, ShouldHaveLength, 2)
			So(err, ShouldBeNil)

		})

		Convey("should be failed", func() {
			mock.ExpectQuery("^SELECT (.+) FROM posts$").
				WillReturnError(fmt.Errorf("list post error"))

			items, err := dao.List()
			So(items, ShouldBeNil)
			So(err.Error(), ShouldContainSubstring, "list post error")
		})
	})
}
```

复制代码

与 Docker 集成
-----------

虽然我们可以采用 Mock 的方式屏蔽掉某些服务，但是还是存在某些服务比较复杂，很难 mock（如 MongoDB） ，而且有时我们确实想和依赖的服务做某些集成测试。

此时我们可以用 Docker 来快速构建我们的测试依赖环境，并且用完即释放、非常高效，下面就是一个包含 MongoDB 的 Dockerfile：

```yaml
FROM ubuntu:16.04
RUN apt-get update && apt-get install -y libssl1.0.0 libssl-dev gcc

RUN mkdir -p /data/db /opt/go/ /opt/gopath
COPY mongodb/bin/* /usr/local/bin/

ADD go /opt/go
RUN cp /opt/go/bin/* /usr/local/bin/
ENV GOROOT=/opt/go GOPATH=/opt/gopath

WORKDIR /ws
CMD mongod --fork --logpath /var/log/mongodb.log && GOPROXY=off go test -mod=vendor ./...
```

复制代码

总结
--

本次分享主要从单元测试的重要性入手，依次讲解了官方库 testing、社区测试框架（Testify、GoConvey、GinkGo）、Mock 相关技术、Docker 集成的内容，总结如下：

*   单元测试应该是一个团队共识
    
*   单元测试并不难
    
*   使用 Mock 能够使我们单元测试高效
    
*   应该面向接口编程，方便做 mock （gomock）
    
*   官方库足够优秀，包含表格测试、http 测试相关内容
    
*   社区有很多优秀测试框架，能够让我们更好的实践 BDD 或者 TDD
    
*   Docker 能够适用于更复杂的测试场景
    

参考
--

*   [https://github.com/songjiayang/gotesting](https://github.com/songjiayang/gotesting)  
    
*   [https://golang.org/pkg/testing](https://golang.org/pkg/testing)  
    
*   [https://golang.org/pkg/cmd/go/internal/test](https://golang.org/pkg/cmd/go/internal/test)  
    
*   [https://golang.org/pkg/testing/#hdr-Subtests_and_Sub_benchmarks](https://golang.org/pkg/testing/#hdr-Subtests_and_Sub_benchmarks)  
    
*   [https://golang.org/pkg/net/http/httptest](https://golang.org/pkg/net/http/httptest)  
    
*   [https://github.com/stretchr/testify](https://github.com/stretchr/testify)  
    
*   [https://github.com/onsi/ginkgo](https://github.com/onsi/ginkgo)  
    
*   [https://en.wikipedia.org/wiki/Behavior-driven_development](https://en.wikipedia.org/wiki/Behavior-driven_development)  
    
*   [https://github.com/onsi/gomega](https://github.com/onsi/gomega)  
    
*   [https://github.com/smartystreets/goconvey](https://github.com/smartystreets/goconvey)  
    
*   [https://github.com/golang/mock](https://github.com/golang/mock)  
    
*   [https://github.com/jarcoal/httpmock](https://github.com/jarcoal/httpmock)  
    
*   [https://github.com/DATA-DOG/go-sqlmock](https://github.com/DATA-DOG/go-sqlmock)