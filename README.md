# JieLi-AC695N-I2S
Audio IIS接口(以下简称ALNK)是一个通用的双声道音频接口,用于连接片外的DAC或ADC,连接信号有MCLK, SCLK，LRCK，DATA,原生支持16/24bit数据位宽，对18/20/32位宽的设备可提供兼容支持，目前仅支持IIS输出模式。
ALINK可配置为主机或从机模式。主机模式是指SCLK、LRCK由本模块提供，从机模式则由外部提供时钟。常用采样率为44.1KHZ或48KHZ，可用8K/11.025K/12K/16K/22.05K/24K。三组硬件通道ALINK0_PORTA、 ALINK1_PORTA可选（默认使用ALINK1_PORTA MCLK:PB0 SCLK:PC0 LRCK:PC1 DAT:PC2）,ALINK0_PORTB暂不可用。相关配置寄存器有ALINK_CON0、ALINK_CON3等控制

在SDK中配置表现为相关结构体，可按实际需求配置：
typedef struct {
    iis_isr_cbfun   isr_cbfun; 		///< alink中断的回调函数句柄，不用回调函数则写入NULL，如无中断，句柄无效
    ALINK_PORT      port;         	///< alink端口选择
    u8  soe;                        ///< alink是否使能sclk和lrck
    u8  moe;                        ///< alink是否使能mclk
    u8  dsp;                        ///< alink选择扩展模式
    u32 rate;                       ///< alink采样率
    iis_channel ch[ALINK_CH_MAX];	///< alink通道参数
    u32 frame_len;                  ///< alink每次中断的数据长度
} iis_param;
typedef struct {
    u8  enable;                  	///< alink通道使能
    u8  dir;                    	///< alink方向选择
    u8  bit_wide;               	///< alink数据位宽选择
    u8  ch_md;                  	///< alink模式选择
    s16 *dma_adr;     				///< alink dma地址
} iis_channel;


§1.2  相关程序配置说明
	官方SDK release V0.1.0 中board_ac695x_demo_cfg.h有相关可选配置可直接修改：
#define AUDIO_OUTPUT_WAY            AUDIO_OUTPUT_WAY_DAC_IIS
#define TCFG_IIS_ENABLE                       ENABLE_THIS_MOUDLE //
#define TCFG_IIS_OUTPUT_EN                    ENABLE //
#define TCFG_IIS_OUTPUT_PORT                  ALINK0_PORTA
#define TCFG_IIS_OUTPUT_CH_NUM                1 //0:mono,1:stereo
#define TCFG_IIS_OUTPUT_SR                    48000
#define TCFG_IIS_OUTPUT_DATAPORT_SEL          (BIT(0))
	其中程序配置初始化函数由int audio_link_open(u8 alink_port, u8 dir)、int iis_open(iis_param *param)配置并注册回调函数void iis0_cb(u8 ch_idx, void *buf, u32 len)，在audio_link.c文件中可查询。
注：程序中默认IIS输出使用主机模式，输入使用从模式，需改变按以下步骤。
 
图1.2.1
同时从机模式由外部提供时钟信号，故配置相关IO为输入：
 
图1.2.2

其中注册传输数据中断回调函数：
alink_hdl->param.isr_cbfun = (alink_port < 2) ? iis0_cb : iis1_cb;
alink中断的回调函数句柄，不用回调函数则写入NULL，如无中断，句柄无效
 
该函数操作IIS缓存数据结构体可通过alink 输出数据的写入接口：
typedef struct _cbuffer {
    u8  *begin;
    u8  *end;
    u8  *read_ptr;
    u8  *write_ptr;
    u8  *tmp_ptr ;
    u32 tmp_len;
    u32 data_len;
    u32 total_len;
    spinlock_t lock;
} cbuffer_t;

 
§1.3 逻辑分析仪输出波形
