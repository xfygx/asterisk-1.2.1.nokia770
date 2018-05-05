#include <stdio.h>
#include <ctype.h>	/* for isalnum */
#include <string.h>
#include <unistd.h>
#include <sys/ioctl.h>
#include <fcntl.h>
#include <sys/time.h>
#include <stdlib.h>
#include <errno.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <gst/gst.h>

#include <sys/types.h>
#include <sys/select.h>
#include <errno.h>

#ifdef __linux
#include <linux/soundcard.h>
#elif defined(__FreeBSD__)
#include <sys/soundcard.h>
#else
#include <soundcard.h>
#endif

#include "asterisk.h"

ASTERISK_FILE_VERSION(__FILE__, "$Revision: 36998 $")

#include "asterisk/lock.h"
#include "asterisk/frame.h"
#include "asterisk/logger.h"
#include "asterisk/channel.h"
#include "asterisk/module.h"
#include "asterisk/options.h"
#include "asterisk/pbx.h"
#include "asterisk/config.h"

#include "asterisk/cli.h"
#include "asterisk/utils.h"
#include "asterisk/causes.h"
#include "asterisk/endian.h"

static char *config = "gstreamer.conf";	/* default config file */

static pthread_t monitor_thread = AST_PTHREADT_NULL;

struct chan_gstreamer_pvt {
	char *type;
	struct ast_channel *owner;
	struct ast_frame read_f;
	char language[20];	
};

static struct chan_gstreamer_pvt gstreamer_default = {
	.type = "GStreamer",
	.owner = NULL,
};

#define GST_TYPE_PCMU "audio/x-mulaw"
#define GST_TYPE_PCMA "audio/x-alaw"
#define GST_TYPE_PCM  "audio/x-raw-int"

static struct ast_channel *gstreamer_request(const char *type, int format, void *data, int *cause);
static int gstreamer_digit(struct ast_channel *c, char digit);
static int gstreamer_text(struct ast_channel *c, const char *text);
static int gstreamer_hangup(struct ast_channel *c);
static int gstreamer_answer(struct ast_channel *c);
static struct ast_frame *gstreamer_read(struct ast_channel *chan);
static int gstreamer_call(struct ast_channel *c, char *dest, int timeout);
static int gstreamer_write(struct ast_channel *chan, struct ast_frame *f);
static int gstreamer_indicate(struct ast_channel *chan, int cond);
static int gstreamer_fixup(struct ast_channel *oldchan, struct ast_channel *newchan);

typedef enum
{
    DTX_OFF = 0,
    DTX_ON = 1
} DTX;

struct t_chan_buf{
	char rec_buffer_cache[4096];
	char rec_buffer[4096];
	int  size;
	int  update_buffer;
};

struct t_chan_buf chan_buf;

static int gst_sk;
static int voc_sk;
static int voc_play_sk;

char gbuf[1024];
GstElement *pipeline_rec = NULL;
GstElement *pipeline_play = NULL;

static const struct ast_channel_tech gstreamer_tech = {
	.type =			"GSTREAMER",
	.description =	"GStreamer Channel Driver",
	.capabilities =	AST_FORMAT_ULAW,
	.requester =	gstreamer_request,
	.send_digit =	gstreamer_digit,
	.send_text =	gstreamer_text,
	.hangup =		gstreamer_hangup,
	.answer =		gstreamer_answer,
	.read =			gstreamer_read,
	.call =			gstreamer_call,
	.write =		gstreamer_write,
	.indicate =		gstreamer_indicate,
	.fixup =		gstreamer_fixup,
};

/*
 *  * split a string in extension-context, returns pointers to malloc'ed
 *   * strings.
 *    * If we do not have 'overridecontext' then the last @ is considered as
 *     * a context separator, and the context is overridden.
 *      * This is usually not very necessary as you can play with the dialplan,
 *       * and it is nice not to need it because you have '@' in SIP addresses.
 *        * Return value is the buffer address.
 *         */
static char *ast_ext_ctx(const char *src, char **ext, char **ctx)
{
	if (ext == NULL || ctx == NULL)
		return NULL;    /* error */
	*ext = *ctx = NULL;
	if (src && *src != '\0')
		*ext = strdup(src);
	if (*ext == NULL)
                return NULL;
	if (1) {
		/* parse from the right */
                *ctx = strrchr(*ext, '@');
		if (*ctx)
			*(*ctx)++ = '\0';
	}
	return *ext;
}

static int gstreamer_digit(struct ast_channel *c, char digit)
{
	return 0;
}

static int gstreamer_text(struct ast_channel *c, const char *text)
{
	return 0;
}
/*
 * handler for incoming calls. Either autoanswer, or start ringing
 */
static int gstreamer_call(struct ast_channel *c, char *dest, int timeout)
{
	ast_log(LOG_NOTICE, "\n");
	return 0;
}

/*
 * remote side answered the phone
 */
static int gstreamer_answer(struct ast_channel *c)
{
	ast_log(LOG_NOTICE, "\n");

	ast_setstate(c, AST_STATE_UP);	
	
	return 0;
}

static int gstreamer_hangup(struct ast_channel *c)
{
	struct chan_gstreamer_pvt *o = c->tech_pvt;
	
	ast_log(LOG_NOTICE, "\n");

	ast_setstate(c, AST_STATE_DOWN);

	c->fds[0] = 0;	
	c->tech_pvt = NULL;
	o->owner = NULL;
	
	return 0;
}

/* used for data coming from the network */
static int gstreamer_write(struct ast_channel *c, struct ast_frame *f)
{
	int ret, length;
	struct sockaddr_in server;

	bzero(&server, sizeof(server));	
	server.sin_family = AF_INET;
	server.sin_port = htons(4951);
	inet_pton(AF_INET, "127.0.0.1", &server.sin_addr);
	
	ret = sendto(voc_play_sk, f->data, f->datalen, MSG_DONTWAIT,  (struct sockaddr *)&server, sizeof(server));

	//ast_log(LOG_NOTICE, "ret = %d\n", ret);
	
	return 0;
}

static struct ast_frame *gstreamer_read(struct ast_channel *c)
{
	struct chan_gstreamer_pvt *o = c->tech_pvt;
	struct ast_frame *f = &o->read_f;
	char voicebuf[1024];
	int ret;
		
	ret = recv(voc_sk, voicebuf, sizeof(voicebuf), 0);
	
	//ast_log(LOG_NOTICE, "size = %d\n", ret);	
	
	if (ret > 0){
		memcpy((char *)(f->data), voicebuf, ret);
		f->frametype = AST_FRAME_VOICE;
		f->subclass = AST_FORMAT_ULAW;
		f->datalen = ret;
	}

	return f;
}

static int gstreamer_fixup(struct ast_channel *oldchan, struct ast_channel *newchan)
{
	ast_log(LOG_NOTICE, "\n");
	return 0;
}

static int gstreamer_indicate(struct ast_channel *c, int cond)
{
	ast_log(LOG_NOTICE, "\n");
	return 0;
}

/*
 * allocate a new channel.
 */
static struct ast_channel *gstreamer_new(char *ext, char *ctx, int state)
{
	struct ast_channel *c;
	struct chan_gstreamer_pvt *o =  &gstreamer_default;
	
	c = ast_channel_alloc(1);
	if (c == NULL)
		return NULL;
	c->tech = &gstreamer_tech;
	snprintf(c->name, sizeof(c->name), "GStreamer");
	c->type = o->type;
	c->fds[0] = voc_sk;
	c->nativeformats = AST_FORMAT_ULAW;
	c->readformat = AST_FORMAT_ULAW;
	c->writeformat = AST_FORMAT_ULAW;
	c->tech_pvt = o;

	if (!ast_strlen_zero(ctx))
		ast_copy_string(c->context, ctx, sizeof(c->context));
	if (!ast_strlen_zero(ext))
		ast_copy_string(c->exten, ext, sizeof(c->exten));
	if (!ast_strlen_zero(o->language))
		ast_copy_string(c->language, o->language, sizeof(c->language));

	o->owner = c;
	ast_setstate(c, state);
	ast_update_use_count();
	if (state != AST_STATE_DOWN) {
		if (ast_pbx_start(c)) {
			ast_log(LOG_WARNING, "Unable to start PBX on %s\n", c->name);
			ast_hangup(c);
			o->owner = c = NULL;
			/* XXX what about the channel itself ? */
			/* XXX what about usecnt ? */
		}
	}
	return c;
}

static struct ast_channel *gstreamer_request(const char *type,
	int format, void *data, int *cause)
{
	return NULL;
}

/*! \brief  do_monitor: The AD monitoring thread ---*/
static void *do_monitor(void *data)
{

	return NULL;
}

/*! \brief  restart_monitor: Start the channel monitor thread ---*/
static int restart_monitor(void)
{
	/* Start a new monitor */
	if (ast_pthread_create(&monitor_thread, NULL, do_monitor, NULL) < 0) {
		ast_log(LOG_ERROR, "Unable to start monitor thread.\n");
		return -1;
	}
	
	return 0;
}

/* for recoder*/
static gboolean cb_have_data (GstPad *pad, GstBuffer *buffer, gpointer u_data)
{
	guint8 *data = buffer->data;
	guint size = buffer->size;
	int ret;
	struct sockaddr_un sun;

	//ast_log(LOG_NOTICE, "%x %x %x %x %x %x %d\n", data[0], data[1], data[2], data[3], data[4], data[5], size);
	sun.sun_family = AF_UNIX;
	strcpy(sun.sun_path, "/var/run/fxsvoc");
	ret = sendto(gst_sk, data, size, MSG_DONTWAIT,  (struct sockaddr *)&sun, sizeof(sun));
	
	return TRUE;
}


static int create_pipeline_rec(void)
{
    GstElement *src = NULL;
    GstElement *sink = NULL;
    GstElement *filter = NULL;
    GstCaps *caps = NULL;
	GstPad *pad;

    pipeline_rec = gst_pipeline_new("pipeline");

    /* create elements */
	src = gst_element_factory_make("dsppcmsrc", "source");
	g_object_set(G_OBJECT (src), 
		"blocksize", "160",//DEFAULT_REC_BLOCKSIZE, 
		"dtx", DTX_OFF,
		NULL);
	
	filter = gst_element_factory_make("capsfilter", "filter");
	caps = gst_caps_new_simple(GST_TYPE_PCMU, "rate", G_TYPE_INT, 8000, \
	                       "channels", G_TYPE_INT, 1, NULL);
	g_object_set(G_OBJECT(filter), 
		"caps", caps,
		NULL);

	sink = gst_element_factory_make("fakesink", "sink");
	//g_object_set(G_OBJECT(sink), "location", "tmp.u", NULL);

	pad = gst_element_get_pad (sink, "sink");
	gst_pad_add_buffer_probe (pad, G_CALLBACK (cb_have_data), NULL);
	gst_object_unref (pad);

    if (!src || !pipeline_rec)
    {
        ast_log(LOG_NOTICE, "Could not create GstElement!\n");
        return -1;
    }

	if (!filter){
		ast_log(LOG_NOTICE, "Could not create filter GstElement!\n");
		return -1;
	}
	gst_bin_add_many(GST_BIN(pipeline_rec), src, filter, sink, NULL);

	if (!gst_element_link_many (src, filter, sink, NULL)){
		ast_log(LOG_NOTICE, "gst_element_link failed for src, filter and sink!\n");
		return -1;
	}

	ast_log(LOG_NOTICE, "recording");
    gst_element_set_state(GST_ELEMENT(pipeline_rec), GST_STATE_PLAYING);

	return 0;
}


static int create_pipeline_play(void)
{
	GstElement *src = NULL;
	//GstElement *mulaw = NULL;	
	GstElement *sink = NULL;
	GstElement *filter = NULL;

	GstCaps *caps = NULL;
	GstPad *pad;
	
	pipeline_play = gst_pipeline_new("pipeline_play");

	src = gst_element_factory_make("udpsrc", "src");
	g_object_set(G_OBJECT(src),
		"buffer-size", 1024,
		NULL);
	//mulaw = gst_element_factory_make("mulawdec ", "mulaw");

	filter = gst_element_factory_make("capsfilter", "filter");
	caps = gst_caps_new_simple(
	                GST_TYPE_PCMU,
	                "rate", G_TYPE_INT, 8000,
	                //"signed", G_TYPE_BOOLEAN, 1,
	                "channels", G_TYPE_INT, 1,
	                //"endianness", G_TYPE_INT, 1234,
	                //"width", G_TYPE_INT, 16,
	                //"depth", G_TYPE_INT, 16,
	                NULL);
	g_object_set(G_OBJECT(filter), 
		"caps", caps,
		NULL);		
	gst_caps_unref(caps);
	
	sink = gst_element_factory_make("dsppcmsink", "sink");

	gst_bin_add_many(GST_BIN(pipeline_play), src, filter, sink, NULL);
	
	if (!gst_element_link_many (src, filter, sink, NULL)){
		ast_log(LOG_NOTICE, "gst_element_link failed for src, filter and sink!\n");
		return -1;
	}

	ast_log(LOG_NOTICE, "playing");
	gst_element_set_state(GST_ELEMENT(pipeline_play), GST_STATE_PLAYING);

	return 0;
}

int gstreamer_init(void)
{
	struct sockaddr_un unaddr;
	guint minor = 0, major = 0, micro = 0, nano = 0;
	
	/* initialise gst */
	gst_init(NULL, NULL);
	gst_version (&major, &minor, &micro, &nano);

	chan_buf.size = 0;
	chan_buf.update_buffer =0;
	
	create_pipeline_rec();
	create_pipeline_play();

	/* receive data from cb_have_data() */
	voc_sk = socket(AF_UNIX, SOCK_DGRAM, 0);
	if (voc_sk == -1){
		ast_log(LOG_NOTICE, "opening voc_sk sockeet: %s\n", strerror(errno));
		return -1;
	}

	unaddr.sun_family = AF_UNIX;
	strcpy(unaddr.sun_path, "/var/run/fxsvoc");
	unlink(unaddr.sun_path);
	if (0 > bind(voc_sk, (struct sockaddr *)&unaddr, sizeof(unaddr))) {
		ast_log(LOG_NOTICE, "binding voc_sk socket: %s\n", strerror(errno));
		return -1;
	}

	voc_play_sk = socket(PF_INET, SOCK_DGRAM, 0);
	if (voc_play_sk == -1){
		ast_log(LOG_NOTICE, "opening voc_play_sk sockeet: %s\n", strerror(errno));
		return -1;
	}

	/* send data to voc_sk thought gst_sk */
	gst_sk = socket(AF_UNIX, SOCK_DGRAM, 0);
	if (gst_sk == -1){
		ast_log(LOG_NOTICE, "opening gst_sk sockeet: %s\n", strerror(errno));
		return -1;
	}

	unaddr.sun_family = AF_UNIX;
	strcpy(unaddr.sun_path, "/var/run/gstvoc");
	unlink(unaddr.sun_path);
	if (0 > bind(gst_sk, (struct sockaddr *)&unaddr, sizeof(unaddr))) {
		ast_log(LOG_NOTICE, "binding gst_sk socket: %s\n", strerror(errno));
		return -1;
	}

	gstreamer_default.owner = NULL;
	gstreamer_default.read_f.data = gbuf;	
	return 0;
}

int gstreamer_close(void)
{

	return 0;
}

static int console_hangup(int fd, int argc, char *argv[])
{
	struct chan_gstreamer_pvt *o = &gstreamer_default;

	ast_setstate(o->owner, AST_STATE_DOWN);	

	gst_element_set_state (pipeline_rec, GST_STATE_NULL);
	gst_element_set_state (pipeline_play, GST_STATE_NULL);	

	gst_object_unref (GST_OBJECT (pipeline_rec));
	gst_object_unref (GST_OBJECT (pipeline_play));	
	
	return 0;
}

static char hangup_usage[] =
"Usage: hangup\n"
"       Hangs up any call currently placed on the console.\n";

	
static int console_dial(int fd, int argc, char *argv[])
{
	char *s = NULL, *mye = NULL, *myc = NULL;

	if (argc != 1 && argc != 2)
		return RESULT_SHOWUSAGE;

	/* if we have an argument split it into extension and context */
	if (argc == 2)
		s = ast_ext_ctx(argv[1], &mye, &myc);
	/* supply default values if needed */
	if (mye == NULL){
		ast_log(LOG_NOTICE, "mye is NULL");
		return -1;
	}
	if (myc == NULL){
		ast_log(LOG_NOTICE, "myc is NULL");
		return -1;
	}
	if (ast_exists_extension(NULL, myc, mye, 1, NULL)) {
		gstreamer_new(mye, myc, AST_STATE_RINGING);
	} else
		ast_cli(fd, "No such extension '%s' in context '%s'\n", mye, myc);
	if (s)
		free(s);
	return RESULT_SUCCESS;
}

static char dial_usage[] =
"Usage: dial [extension[@context]]\n"
"       Dials a given extensison (and context if specified)\n";

static struct ast_cli_entry myclis[] = {
	{ { "dial", NULL }, console_dial, "Dial an extension on the console", dial_usage },
	{ { "hangup", NULL }, console_hangup, "Hangup a call on the console", hangup_usage }	
};

int load_module(void)
{
	struct ast_config *cfg;

	 ast_log(LOG_NOTICE, "load chan_gstreamer.so\n");
	 
	/* load config file */
	cfg = ast_config_load(config);
	if (cfg != NULL) {
		 ast_log(LOG_NOTICE, "load config gstreamer.conf\n");		
	}
	else{
		 ast_log(LOG_NOTICE, "Unable to load config gstreamer.conf\n");
	}

	if (gstreamer_init() != 0){
		ast_log(LOG_NOTICE, "Unable to open kinds of device\n");
		return -1;
	}

	if (ast_channel_register(&gstreamer_tech)) {
		ast_log(LOG_ERROR, "Unable to register channel class \n");
		/* XXX should cleanup allocated memory etc. */
		return -1;
	}

	/* And start the monitor for the first time */
	restart_monitor();

	ast_log(LOG_NOTICE, "load module complete!\n");

	ast_cli_register_multiple(myclis, sizeof(myclis)/sizeof(struct ast_cli_entry));	
	return 0;
}

int unload_module()
{
	ast_log(LOG_NOTICE, "unload chan_gstreamer module\n");

	gst_element_set_state (pipeline_rec, GST_STATE_NULL);
	gst_element_set_state (pipeline_play, GST_STATE_NULL);	

	gst_object_unref (GST_OBJECT (pipeline_rec));
	gst_object_unref (GST_OBJECT (pipeline_play));	

	return 0;
}

char *description()
{
	return (char *)gstreamer_tech.description;
}

int usecount()
{
	return 0;
}

char *key()
{
	return ASTERISK_GPL_KEY;
}

