# fasthttp-rendering-jet-template

<pre>
//globals
import(
        "github.com/CloudyKit/jet"
        "github.com/patrickmn/go-cache"
        "github.com/tdewolff/minify"
        ......
)
var myCache = cache.New(30 * time.Minute, time.Minute)
var myCompress = NewCompress()
var myConfig Config
var views = jet.NewHTMLSet("./views")

//fasthttp handler
func home(ctx *fasthttp.RequestCtx) {
  locale:=GetUserLocale(ctx)
  ck := fmt.Sprintf("%s_home", locale)   //ck: cache key
	if RenderCache(ctx, ck) {
		return
	}
	p := Page{
		Locale:   locale,
		Title:    LS(locale, "Home"),  //LS: func to get translated string
	}
  ......
	m := jet.VarMap{}
	m.Set("LS", LS)	  //use LS in jet template
	m.Set("news", GetPostList(co, "news", 10))
	m.Set("products", GetPostList(co, "products", 10))
  ......
	Render(ctx, "public/home", ck, m, p)
}

func Render(ctx *fasthttp.RequestCtx, t, ck string, m jet.VarMap, d interface{}) {
	vw, err := views.GetTemplate(t)
	if err != nil {
		RenderError(ctx, "Template error:" + err.Error())
		return
	}
	ctx.SetContentType("text/html")
	cache := Cache{Type: "text/html"}
	buf := bytes.Buffer{}
	err = vw.Execute(&buf, m, d)
	if err != nil {
		RenderError(ctx, "Render error:" + err.Error())
		return
	}
	var body []byte
	if myConfig.Debug {
		body = buf.Bytes()
	} else {
		body = myCompress.MinifyBytes("text/html", buf.Bytes())
	}
	gzip := strings.Contains(string(ctx.Request.Header.Peek("Accept-Encoding")), "gzip")
	gziped := false
	if gzip && len(body) > 512 {
		zbuf := Gzip(body)
		n := len(zbuf)
		if n > 0 && n < len(body) {
			gziped = true
			ctx.Response.Header.Set("Content-Encoding", "gzip")
			ctx.SetBody(zbuf)
			if ck != "" {
				cache.Gzip = true
				cache.Body = make([]byte, n)
				copy(cache.Body, zbuf)
			}
		}
	}
	if !gziped {
		ctx.SetBody(body)
		if ck != "" && len(body) > 0 {
			cache.Body = make([]byte, len(body))
			copy(cache.Body, body)
		}
	}
	if ck != "" && len(cache.Body) > 0 {
		cache.Etag = HashStr(cache.Body)
		ctx.Response.Header.Set("ETag", cache.Etag)
		myCache.Add(ck, cache, 30 * time.Minute)
	}
	VisitLog(ctx, "Render", fasthttp.StatusOK, len(ctx.Response.Body()))
}

func RenderError(ctx *fasthttp.RequestCtx, msg string) {
	log.Println(msg)
	if myConfig.Debug {
		ctx.SetBodyString(msg)
	} else {
		ctx.Error("Server Error", 500)
	}
}

type Cache struct {
	Etag string
	Type string
	Body []byte
	Gzip bool
}

func RenderCache(ctx *fasthttp.RequestCtx, ck string) bool {
	if ck == "" || myConfig.Debug {
		return false
	}
	cd, ok := myCache.Get(ck)
	if !ok {
		return false
	}
	ce := cd.(Cache)
	etag := string(ctx.Request.Header.Peek("If-None-Match"))
	if etag == ce.Etag {
		ctx.NotModified()
		if ce.Type == "text/html" {
			VisitLog(ctx, "Cache", fasthttp.StatusNotModified, 0)
		}
	} else {
		gzip := strings.Contains(string(ctx.Request.Header.Peek("Accept-Encoding")), "gzip")
		if !gzip && ce.Gzip {
			buf, err := Gunzip(ce.Body)
			if err != nil {
				RenderError(ctx, "Gunzip error: " + err.Error())
				return false
			}
			ctx.SetBody(buf)
			ctx.Response.Header.Set("Content-Length", strconv.Itoa(len(buf)))
		} else {
			ctx.SetBody(ce.Body)
			ctx.Response.Header.Set("Content-Length", strconv.Itoa(len(ce.Body)))
			if ce.Gzip {
				ctx.Response.Header.Set("Content-Encoding", "gzip")
			}
		}
		ctx.SetContentType(ce.Type)
		ctx.Response.Header.Set("ETag", ce.Etag)
		if ce.Type == "text/html" {
			VisitLog(ctx, "Cache", fasthttp.StatusOK, len(ce.Body))
		}
	}
	return true
}

//some funcs used in Render func
type Compress struct {
	minifier *minify.M
}

func NewCompress() *Compress {
	m := minify.New()
	m.AddFunc("text/css", css.Minify)
	m.AddFunc("text/html", html.Minify)
	m.AddFunc("text/javascript", js.Minify)
	m.AddFunc("image/svg+xml", svg.Minify)
	m.AddFuncRegexp(regexp.MustCompile("[/+]json$"), json.Minify)
	m.AddFuncRegexp(regexp.MustCompile("[/+]xml$"), xml.Minify)
	return &Compress{
		minifier: m,
	}
}

func (c *Compress) MinifyBytes(ct string, src []byte) []byte {
	dst, err := c.minifier.Bytes(ct, src)
	if err == nil {
		return dst
	}
	return src
}

import "github.com/klauspost/compress/gzip"   //this is faster than the default gzip

func Gzip(data []byte) []byte {
	var buf bytes.Buffer
	w := gzip.NewWriter(&buf)
	defer w.Close()
	w.Write(data)
	w.Flush()
	return buf.Bytes()
}

func Gunzip(data []byte) ([]byte, error) {
	buf := bytes.NewBuffer(data)
	r, err := gzip.NewReader(buf)
	if err != nil {
		return nil, err
	}
	defer r.Close()
	ud, _ := ioutil.ReadAll(r)
  if err != nil {
		return nil, err
	}
	return ud, nil
}

import "github.com/OneOfOne/xxhash"   //xxhash is the fastest hash

func Hash(data []byte) uint64 {
	return xxhash.Checksum64(data)
}

func HashStr(data []byte) string {
	return fmt.Sprintf("%x", Hash(data))[2:]
}
</pre>
