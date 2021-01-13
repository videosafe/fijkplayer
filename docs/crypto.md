# ijkplayer 自定义协议播放加密内容 Android

想对播放的音视频进行加密，防止资源被盗用，该怎么办呢？
这篇文章从自定义协议的角度来提供一中实现思路。在 ijkplayer 的基础上，通过实现自定义协议对文件进行解密。边解边播，以此为基础，还可以实现在线资源边下载边解密边播放。
结合 ijkplayer 源码阅读本文效果最佳。
FFmpeg 文件协议
ffmpeg 中定义了 URLProtocol， 对接入 ffmpeg 中的各种协议进行了统一的抽象，这里可以理解为 C++ 中的抽象基类，并基于此抽象协议实现了 file、http、ftp、cache 等许多不同的具体文件传输协议。
而在 libavformat 模块中的 avio.c aviobuf.c 两个文件则是在 ffmpeg 对文件传输协议抽象的基础上进行的封装，使对文件的操作能够在 ffmpeg 中其他对文件协议细节不关心的地方使用。这就好比是面向对象中的模版模式。
ffmpeg 中的文件协议简单讲就是这样，更深入复杂的我也还没具体研究，本文中也用不到。
URLProtocol 的部分代码如下，这里我只留下了最常用的一部分。URLProtocol 中定义了很多函数指针，在 avio.c 的模板模式代码中用到。
typedef struct URLProtocol {
    const char *name;
    int     (*url_open)( URLContext *h, const char *url, int flags);
    int     (*url_read)( URLContext *h, unsigned char *buf, int size);
    int     (*url_write)(URLContext *h, const unsigned char *buf, int size);
    int64_t (*url_seek)( URLContext *h, int64_t pos, int whence);
    int     (*url_close)(URLContext *h);
    int priv_data_size;
    const AVClass *priv_data_class;
} URLProtocol;
通过实现 URLProtocol 中的函数指针，就完成了一个可以用在 ffmpeg 中的文件协议。比如我找了一番 ffmpeg 中一个比较简单的文件协议，从实现上可以看出这个 bluray 在ffmpeg 中是一个只读协议。
const URLProtocol ff_bluray_protocol = {
    .name            = "bluray",
    .url_close       = bluray_close,
    .url_open        = bluray_open,
    .url_read        = bluray_read,
    .url_seek        = bluray_seek,
    .priv_data_size  = sizeof(BlurayContext),
    .priv_data_class = &bluray_context_class,
};
ffmpeg 中通过一个列表保存了所有实现了的文件协议，进行各种文件操作的第一步，就是先根据 url（filename）找到对应的 URLProtocol。这段代码中通过匹配 url 中的 scheme 字符串，找到并返回对应的 protocol 指针。
static const struct URLProtocol *url_find_protocol(const char *filename)
{
    // ......
    const URLProtocol **protocols = ffurl_get_protocols(NULL, NULL);
    if (!protocols)
        return NULL;
    for (i = 0; protocols[i]; i++) {
            const URLProtocol *up = protocols[i];
        if (!strcmp(proto_str, up->name)) {
            av_freep(&protocols);
            return up;
        }
    }
    // ......
}
如果对 ffmpeg 有一定的了解，可能会知道  ffmpeg 中的 Codec 还有 avcodec_register。FFmpeg 通过接口 avcodec_register 在运行时动态添加编解码器。但是却没有类似的 avformat_register 或者 av_urlprotocol_register 来实现运行时动态注册自定义的 protocol。为什么不实现这个的原因没有研究过，所以不能瞎吹。但是我们可以改 ffmpeg 源代码呀。
ijkplayer 的作者就改了 ffmpeg ，新增了几个 URLProtocol 并加入到了默认的 protocols 数组中，这些新增的 protocols 在 ffmpeg 源码中默认是空的实现。但是在 ffmpeg 中预留了接口可以在 ijkplayer 中使用后期实现的 protocol 替换这个空的 protocol。相当于添加了作用类似于 av_urlprotocol_register 但是有一定限制的接口。
ijkmediadatasource 协议实现
在 ijkplayer 项目的 ijkmediadatasource.c 源文件中，实现了一种 URLProtocol
URLProtocol ijkimp_ff_ijkmediadatasource_protocol = {
    .name                = "ijkmediadatasource",
    .url_open2           = ijkmds_open,
    .url_read            = ijkmds_read,
    .url_seek            = ijkmds_seek,
    .url_close           = ijkmds_close,
    .priv_data_size      = sizeof(Context),
    .priv_data_class     = &ijkmediadatasource_context_class,
};
在每个具体的函数指针实现中，又通过 J4A 生成的胶水代码去调用到 java 层的代码。
进一步解释一下， setDataSource 的时候调用到这一块代码，callback 已经是一个具体的 java 对象了，并且是一个 `tv/danmaku/ijk/media/player/misc/IMediaDataSource` 接口的具体实现。 
IjkMediaPlayer_setDataSourceCallback(JNIEnv *env, jobject thiz, jobject callback) {
    nativeMediaDataSource = jni_set_media_data_source(env, thiz, callback);
    snprintf(uri, sizeof(uri), "ijkmediadatasource:%"PRId64, nativeMediaDataSource);
    retval = ijkmp_set_data_source(mp, uri);
}
jni_set_media_data_source 函数新建一个 NDK 环境对 callback 这个 jobject 的全局引用，并把引用作为 intptr_t 类型保存在 IjkMediaPlayer 的 mNativeMediaDataSource 字段中，方便后面再次使用以及最后需要close 是用到。 通过这样的转换之后，url 进入 ffmpeg  avformat 逻辑中就成了 “ijkmediadatasource:2234234290" 这样的形式。 再通过 url_find_protocol 找到  ijkimp_ff_ijkmediadatasource_protocol ，所有的对这个文件协议的操作又回到 ijkplayer 的代码中了。
在 ijkmds_open  中，只需要把 intptr_t 的变量转换成 jobject 就行，不必实际去打开某个文件。 ijkmds_read 、 ijkmds_seek  函数通过 J4A 去调用  IMediaDataSource 接口的 readAt 方法。 readAt 方法多了 pos 参数，所以 read 和 seek 都可以通过 readAt 实现， ijkmds_close  调用 IMediaDataSource 接口的 close 方法，并释放 NDK 环境中的 jobject 全局引用。
通过这一层套一层的接口定义实现、struct 函数指针定义实现， java 代码中设置的 dataSource 在 ijkplayer 、ffmpeg 中打了个转之后最终又回到 java 代码中。
jni4android 自动生成代码
 jni4android 是 B 站出品的开源项目 https://github.com/bilibili/jni4android。能够根据简单的 java 接口描述代码生成 ndk 胶水代码，方便在 ndk c/c++ 环境中调用 java 代码。
ijkmediadatasource.c 源文件中 JNI 相关调用的代码，都是 j4a 自动生成的。按照 jni4android 中 readme 编译好 j4a 之后，结合 ijkplayer/ijkmedia/j4a  中的 makefile 文件，就可以快速掌握 j4a 在 ijkplayer 中的作用和使用方法了。
加密和解密
前面代码分析中解释了 ijkplayer 中自定义文件协议到底是怎么一回事，以及其中的函数调用过程怎么样的。
分析完了，终于可以开始代码敲起来。我们先对一个视频文件加密，然后实现解密的文件协议并在播放中使用。
这里采用古老的凯撒加解密算法，加密过程就是每个 byte 加个数字，解密过程就是每个 byte 减去个数字。相当简单，很容易暴力破解，这里仅作为示例参考。
加密过程如下(截取部分代码)：
func main() {
    ibuf := bufio.NewReader(inputfile)
    obuf := bufio.NewWriter(outputfile)
    buf := make([]byte, 128)
    for {
        n, err := ibuf.Read(buf)
        if err == nil {
            for index := 0; index < n; index++ {
                buf[index] = buf[index] + byte(cck)
            }
            obuf.Write(buf[:n])
        } else if err == io.EOF {
            break
        }
    }
    obuf.Flush()
}
解密的 java IMediaDataSource 实现如下，这里只给出关键部分，其他的和项目中 FileMediaDataSource 一样。
public class CCFileMediaDataSource implements IMediaDataSource {
    @Override
    public int readAt(long position, byte[] buffer, int offset, int size) throws IOException {
        if (mFile.getFilePointer() != position)
            mFile.seek(position);
        if (size == 0) return 0;
        int s = mFile.read(buffer, 0, size);
        for (int i = 0; i < s; i++) {
            buffer[i] = (byte)(buffer[i] - 10);
        }
        return s;
    }
}
什么情况下会用到 CCFileMediaDataSource 呢？ 简单粗暴，把项目中 IjkVieoView.java 用到 FileMediaDataSource 的地方都改成 CCFileMediaDataSource，其实也没几处，然后别忘了在 demo 中设置选项里选中使用MediaDataSource。
IMediaDataSource 其他实现
RandomAccessFile 实现的 IMediaDataSource， 支持本地保存的文件。
public class FileMediaDataSource implements IMediaDataSource {
    private RandomAccessFile mFile;
    private long mFileSize;
    public FileMediaDataSource(File file) throws IOException {
        mFile = new RandomAccessFile(file, "r");
        mFileSize = mFile.length();
    }
    @Override
    public int readAt(long position, byte[] buffer, int offset, int size) throws IOException {
        if (mFile.getFilePointer() != position)
            mFile.seek(position);
        if (size == 0) return 0;
        return mFile.read(buffer, 0, size);
    }
    @Override
    public long getSize() throws IOException {
        return mFileSize;
    }
    @Override
    public void close() throws IOException {
        mFileSize = 0;
        mFile.close();
        mFile = null;
    }
}
InputStream 实现的 IMediaDataSource，可以用于播放 asset 资源文件
public class StreamDataSource implements IMediaDataSource {
    private InputStream mIs;
    private long mPosition = 0;
    public StreamDataSource(InputStream mIs) {
        this.mIs = mIs;
    }
    @Override
    public int readAt(long position, byte[] buffer, int offset, int size) throws IOException {
        if (size <= 0)
            return size;
        if (mPosition != position) {
            mIs.reset();
            mPosition = mIs.skip(position);
        }
        int length = mIs.read(buffer, offset, size);
        mPosition += length;
        return length;
    }
    @Override
    public long getSize() throws IOException {
        return mIs.available();
    }
    @Override
    public void close() throws IOException {
        if (mIs != null)
            mIs.close();
        mIs = null;
    }
}
  AssetManager assetManager = mContext.getAssets();
  InputStream is = assetManager.open("asset_file_path", AssetManager.ACCESS_RANDOM);
  mIjkMediaPlayer.setDataSource(new StreamDataSource(is));
文中完整代码请在 https://github.com/befovy/blog-codes/tree/master/20191118-ijkplayer-datasource 查看。