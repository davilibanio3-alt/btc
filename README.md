# btc/**
 * @btc-platform/tx-engine
 * Bitcoin Mainnet Transaction Engine
 */

import * as bitcoin from "bitcoinjs-lib";
import * as bip32 from "bip32";
import * as ecc from "tiny-secp256k1";
import { ECPairFactory } from "ecpair";

bitcoin.initEccLib(ecc);

const ECPair = ECPairFactory(ecc);

export const NETWORK = bitcoin.networks.bitcoin;

export interface UTXO {
  txid: string;
  vout: number;
  value: number;
  scriptPubKey?: string;
}

export interface Output {
  address: string;
  value: number;
}

export async function estimateFee(
  feeRate: number,
  inputs: number,
  outputs: number
): Promise<number> {
  const vbytes = inputs * 68 + outputs * 31 + 10;
  return Math.ceil(vbytes * feeRate);
}

export function generateAddress(
  xpub: string,
  index = 0,
  purpose: 84 | 44 | 49 | 86 = 84
): string {

  const node = bip32.BIP32.fromBase58(xpub, NETWORK);

  const child = node.derive(0).derive(index);

  switch (purpose) {

    case 44:
      return bitcoin.payments.p2pkh({
        pubkey: child.publicKey,
        network: NETWORK
      }).address!;

    case 49:
      return bitcoin.payments.p2sh({
        redeem: bitcoin.payments.p2wpkh({
          pubkey: child.publicKey,
          network: NETWORK
        }),
        network: NETWORK
      }).address!;

    case 84:
      return bitcoin.payments.p2wpkh({
        pubkey: child.publicKey,
        network: NETWORK
      }).address!;

    case 86:
      return bitcoin.payments.p2tr({
        internalPubkey: child.publicKey.slice(1, 33),
        network: NETWORK
      }).address!;

    default:
      throw new Error("Unsupported purpose");
  }
}

export async function buildPSBT(
  utxos: UTXO[],
  outputs: Output[],
  fee: number
) {

  const psbt = new bitcoin.Psbt({
    network: NETWORK
  });

  let totalInput = 0;

  for (const utxo of utxos) {

    totalInput += utxo.value;

    psbt.addInput({
      hash: utxo.txid,
      index: utxo.vout,
      witnessUtxo: {
        script: Buffer.from(
          utxo.scriptPubKey || "",
          "hex"
        ),
        value: utxo.value
      }
    });
  }

  let totalOutput = 0;

  for (const output of outputs) {

    totalOutput += output.value;

    psbt.addOutput({
      address: output.address,
      value: output.value
    });
  }

  const change = totalInput - totalOutput - fee;

  if (change < 0) {
    throw new Error("Insufficient balance");
  }

  return {
    psbt,
    change
  };
}

export function signPSBT(
  psbt: bitcoin.Psbt,
  wif: string
) {

  const keyPair = ECPair.fromWIF(
    wif,
    NETWORK
  );

  for (let i = 0; i < psbt.inputCount; i++) {
    psbt.signInput(i, keyPair);
  }

  return psbt;
}

export function finalizePSBT(
  psbt: bitcoin.Psbt
): string {

  psbt.finalizeAllInputs();

  return psbt.extractTransaction().toHex();
}

export async function broadcast(
  txHex: string
): Promise<string> {

  const response = await fetch(
    "https://mempool.space/api/tx",
    {
      method: "POST",
      body: txHex
    }
  );

  if (!response.ok) {
    throw new Error(
      await response.text()
    );
  }

  return response.text();
}

export async function fetchUTXOs(
  address: string
): Promise<UTXO[]> {

  const response = await fetch(
    `https://mempool.space/api/address/${address}/utxo`
  );

  return response.json();
}
