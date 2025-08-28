/* sv_publisher_min.c
 * Compile: gcc sv_publisher_min.c -o sv_pub -liec61850 -lpcap
 * Exec: sudo ./sv_pub eth0
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <signal.h>
#include "iec61850/sv_publisher.h"
#include "hal_time.h"

static volatile int running = 1;
static void sigint_handler(int sig) { running = 0; }

int main(int argc, char** argv) {
    if (argc < 2) {
        printf("Uso: %s <iface>\n", argv[0]);
        return 1;
    }
    const char* iface = argv[1];

    /* Cria o publisher SV na interface desejada */
    SvPublisher publisher = SvPublisher_create(iface);
    if (!publisher) {
        printf("Falha ao criar publisher SV na interface %s\n", iface);
        return 1;
    }

    /* Configura endereço MAC multicast SV (grupo típico) */
    uint8_t dstMac[6] = {0x01,0x0c,0xcd,0x04,0x00,0x01};
    SvPublisher_setDstAddress(publisher, dstMac);

    /* VLAN opcional (recomendado em 9-2LE): PCP=4, VID=0x000 */
    SvPublisher_setVlanPriority(publisher, 4);
    SvPublisher_setVlanId(publisher, 0);

    /* APPID (fluxo de SV). Use um valor único p/ cada fluxo. */
    SvPublisher_setAppId(publisher, 0x4000);

    /* Cria um ASDU (um conjunto de amostras) */
    /* nome do svID, dataset (pode deixar NULL), nº de fasores (4I+4V => 8), sample rate */
    SvPublisher_ASDU asdu = SvPublisher_addASDU(publisher,
        "MU01_SV",             /* svID (aparece no Wireshark) */
        NULL,                  /* data set (opcional) */
        8,                     /* nº de “channels” (ex.: Ia, Ib, Ic, In, Va, Vb, Vc, Vn) */
        4000                   /* smpRate, por ex. 80 amostras/ciclo a 50 Hz => 4000 sps */
    );

    /* confRev exclusivo do fluxo; incremente ao mudar config */
    SvPublisher_ASDU_setConfRev(asdu, 1);

    /* Inicia publisher */
    SvPublisher_start(publisher);
    printf("Publicando SV em %s (APPID=0x%04x, smpRate=%u)\n",
           iface, 0x4000, 4000);

    signal(SIGINT, sigint_handler);

    /* Buffer de amostras INT16 conforme 9-2LE (valores escala/offset via engenharia) */
    int16_t samples[8];

    /* smpCnt é gerenciado pela lib; você só atualiza os valores a cada amostra */
    uint32_t period_us = 1000000 / 4000; /* 4000 amostras/s => 250 us */

    float t = 0.f;
    while (running) {
        /* Gera onda senoidal de teste (ex.: 1 A/1 V em pu) */
        /* Ajuste escala conforme seu mapeamento para INT16 */
        float s = sinf(2.0f * 3.1415926f * 50.0f * t); /* 50 Hz */
        samples[0] = (int16_t)(s * 10000); /* Ia */
        samples[1] = (int16_t)(s * 10000); /* Ib */
        samples[2] = (int16_t)(s * 10000); /* Ic */
        samples[3] = 0;                    /* In */
        samples[4] = (int16_t)(s * 10000); /* Va */
        samples[5] = (int16_t)(s * 10000); /* Vb */
        samples[6] = (int16_t)(s * 10000); /* Vc */
        samples[7] = 0;                    /* Vn */

        /* Publica uma amostra (a lib cuida do smpCnt/BER/ASDU) */
        SvPublisher_ASDU_setSamples(asdu, samples);

        /* Envia */
        SvPublisher_publish(publisher);

        Thread_sleep(period_us / 1000); /* em ms */
        t += period_us / 1e6f;
    }

    SvPublisher_stop(publisher);
    SvPublisher_destroy(publisher);
    return 0;
}
