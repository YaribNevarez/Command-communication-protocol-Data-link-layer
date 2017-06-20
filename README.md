# Command communication protocol (Data-link-layer)

The command communication protocol is developed to serve as an interface between devices of different architecures. This object is a GUI communicator, it implements the command protocol layer for a proper serial communication, and it provides the interface or function members in a higher level software layer. The specific communication commands are implemented and used in this layer class. This definition is in guicom.h.

```C
/*
 * GUICom.h
 *
 *  Created on: 3 de feb. de 2017
 *      Author: Yarib Nevárez
 */

#ifndef SRC_GUICOM_H_
#define SRC_GUICOM_H_

#include "xil_types.h"

typedef enum
{
    TRACE_0 = 0,
    TRACE_1,
    TRACE_2,
    TRACE_3,
    TRACE_4,
    TRACE_5,
    TRACE_6,
    TRACE_7,
    TRACE_8,
    TRACE_9,
    TRACE_ALL = 0xFF
} GUITrace;

typedef uint8_t (* GUIProgressCallback)(void * data, uint32_t progress, uint32_t total);

typedef struct //class
{
    void    (* clearTrace) (GUITrace trace);
    uint8_t (* plotSamples)(GUITrace trace, double * array, uint32_t length);
    void    (* setVisible) (GUITrace trace, uint8_t visible);
    void    (* setTime)    (GUITrace trace, double time);
    void    (* setStepTime)(GUITrace trace, double time);
    void    (* textMsg)    (uint8_t id, char * msg);
    void    (* setProgressCallback)(GUIProgressCallback function, void * data);
} GUICom;

GUICom * GUICom_instance(void);

#endif /* SRC_GUICOM_H_ */
```

In this class definition it be seen the function member names, and it can be inferred their tasks. These function members implement the mechanisms for command communication by the usage of the serial port instance.
In the next lines is exhibited the source code of the GUI communicator.

```C
/*
 * GUICom.c
 *
 *  Created on: 3 de feb. de 2017
 *      Author: Yarib Nevárez
 */

#include "guicom.h"
#include "poxi_hardware.h"
#include "miscellaneous.h"
#include "string.h"

static void    GUICom_clearTrace (GUITrace trace);
static uint8_t GUICom_plotSamples(GUITrace trace, double * array, uint32_t length);
static void    GUICom_setVisible (GUITrace trace, uint8_t visible);
static void    GUICom_setTime    (GUITrace trace, double time);
static void    GUICom_setStepTime(GUITrace trace, double time);
static void    GUICom_textMsg    (uint8_t id, char * msg);
static void    GUICom_setProgressCallback(GUIProgressCallback function, void * data);

static GUIProgressCallback progressCallback = NULL;
static void * progressCallbackData = NULL;

static uint8_t GUICom_doubleBulkLength = 16;

#define CMD_CLEAR         0x00
#define CMD_PLOT          0x01
#define CMD_SET_VISIBLE   0x02
#define CMD_SET_STEP_TIME 0x03
#define CMD_SET_TIME      0x04
#define CMD_TEXT_MSG      0x05

static GUICom GUICom_obj = { GUICom_clearTrace,
                             GUICom_plotSamples,
                             GUICom_setVisible,
                             GUICom_setTime,
                             GUICom_setStepTime,
                             GUICom_textMsg,
                             GUICom_setProgressCallback };

GUICom * GUICom_instance(void)
{
    return & GUICom_obj;
}

static void    GUICom_clearTrace(GUITrace trace)
{
    uint8_t cmd[] = {CMD_CLEAR, 0};
    cmd[1] = trace;
    SerialPort_instance()->sendFrameCommand(cmd, sizeof(cmd), NULL, 0);
}

static uint8_t GUICom_plotSamples(GUITrace trace, double * array, uint32_t length)
{
    uint8_t rc;
    uint8_t  cmd[] = {CMD_PLOT, 0, 0};
    uint32_t i, len;

    cmd[1] = trace;

    rc = array != NULL;

    if (rc)
    {
        for (i = 0; i < length && rc; i += len)
        {
            if(i + GUICom_doubleBulkLength < length)
            {
                len = GUICom_doubleBulkLength;
            }
            else
            {
                len = length - i;
            }
            cmd[2] = len;
            SerialPort_instance()->sendFrameCommand(cmd,
                                                    sizeof(cmd),
                                                    (uint8_t *) &array[i],
                                                    sizeof(double) * len);

            if (progressCallback != NULL)
            {
                rc = progressCallback(progressCallbackData, i + len, length);
            }
            delay_ms(50);
        }
    }

    return rc;
}

static void    GUICom_setVisible (GUITrace trace, uint8_t visible)
{
    uint8_t cmd[] = {CMD_SET_VISIBLE, 0, 0};
    cmd[1] = trace;
    cmd[2] = visible;
    SerialPort_instance()->sendFrameCommand(cmd, sizeof(cmd), NULL, 0);
}

static void    GUICom_setTime    (GUITrace trace, double time)
{
    uint8_t cmd[] = {CMD_SET_TIME, 0};
    cmd[1] = trace;
    SerialPort_instance()->sendFrameCommand(cmd, sizeof(cmd), (uint8_t *)&time, sizeof(double));
}

static void    GUICom_setStepTime(GUITrace trace, double time)
{
    uint8_t cmd[] = {CMD_SET_STEP_TIME, 0};
    cmd[1] = trace;
    SerialPort_instance()->sendFrameCommand(cmd, sizeof(cmd), (uint8_t *)&time, sizeof(double));
}

static void    GUICom_textMsg    (uint8_t id, char * msg)
{
    uint8_t cmd[] = {CMD_TEXT_MSG, 0, 0};
    cmd[1] = id;
    cmd[2] = strlen(msg);
    SerialPort_instance()->sendFrameCommand(cmd, sizeof(cmd), (uint8_t *)msg, sizeof(char) * strlen(msg));
}

static void    GUICom_setProgressCallback(GUIProgressCallback function, void * data)
{
    progressCallback = function;
    progressCallbackData = data;
}
```

The protocol communication is a layer present in all the communicating points, in this case this layer is present in the ZYNQ device and in the Java application in the desktop system. This based command communication must match in all the systems. The command send by a transmitting point must be understood with exactly the same meaning in all the receivers systems.
All the commands have as parameter the desired trace to affect in Java GUI, and the values of times and points are double precision numbers (64 bits). The following table enlist the commands implemented in this layer.


The next code illustrates the usage of the GUI communicator layer, it reduces the effort of the application by performing the tasks of command makeup and communication.

```C
#define NUM_SAMPLES 3

static GUICom * Poxi_JavaGUICom = GUICom_instance();
static double GUISampleBuffer[NUM_SAMPLES];
static char    msg[120];

GUISampleBuffer[0] = 3.14159265;
GUISampleBuffer[1] = 0.0;
GUISampleBuffer[2] = -3.14159265;

Poxi_JavaGUICom->clearTrace(TRACE_ALL);
Poxi_JavaGUICom->setStepTime(TRACE_ALL, DSP_SAMPLE_TIME);
Poxi_JavaGUICom->plotSamples(TRACE_0, GUISampleBuffer, NUM_SAMPLES);

sprintf(msg," Heartbeat rate = %.2f bpm ", fundFrec * 60);
Poxi_JavaGUICom->textMsg(0,msg);
```

Best regards,

-Yarib Nevárez
