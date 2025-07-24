

### Étape 1 : Dépendances nécessaires
Pour intégrer et tester notre décodeur/encodeur Bech32 personnalisé après une réinstallation de MiningCore, voici les dépendances nécessaires :

#### Dépendances C# (dans `BitcoinUtils.cs`)
- `System.Text` : Pour utiliser `StringBuilder` dans `EncodeBech32`.
- `System` : Pour `FormatException` et les types de base comme `byte`.

Ces dépendances doivent être ajoutées au début de `BitcoinUtils.cs` si elles ne sont pas déjà présentes.

#### Dépendances pour les tests (Python)
- **Python 3** : Pour exécuter le script de test qui valide le décodage Bech32.
- **Bibliothèque Python `bech32`** : Pour comparer les résultats de notre décodeur C# avec une implémentation de référence.

### Étape 2 : Réinstaller les dépendances Python (si vous souhaitez refaire les tests)
Si vous souhaitez refaire les tests pour valider le décodeur Bech32 personnalisé, vous devrez réinstaller Python et la bibliothèque `bech32`. Voici les étapes :

1. **Installer Python 3** :
   - Assurez-vous que Python 3 est installé sur votre serveur Ubuntu :
     ```bash
     sudo apt update
     sudo apt install python3 python3-pip
     ```
   - Vérifiez la version de Python :
     ```bash
     python3 --version
     ```

2. **Installer la bibliothèque `bech32`** :
   - Installez la bibliothèque Python `bech32` :
     ```bash
     pip3 install bech32
     ```
   - Note : Si vous travaillez en tant que `root`, vous pourriez voir un avertissement concernant l'utilisation d'un environnement virtuel. Vous pouvez l'ignorer pour ce cas, ou créer un environnement virtuel si vous préférez.

3. **Recréer le script de test Python** :
   - Créez un fichier `test_bech32.py` pour tester le décodage de l'adresse Nito :
     ```bash
     nano /var/miningcore/test_bech32.py
     ```
   - Collez le script suivant :
     ```python
     import bech32

     # Adresse Nito
     nito_address = "nito1qz4puxme5ukxa9fg484yerazext5l07adz9vhdr"

     # Décoder l'adresse Bech32
     hrp, data = bech32.bech32_decode(nito_address)
     if hrp is None or data is None:
         print("Erreur : Adresse Bech32 invalide")
     else:
         print(f"HRP: {hrp}")
         print(f"WitVersion: {data[0]}")
         # Convertir les données (bits 5) en bits 8
         witness_program = bech32.convertbits(data[1:], 5, 8, False)
         if witness_program is None:
             print("Erreur : Conversion Bech32 invalide")
         else:
             print(f"Decoded (hex): {bytes(witness_program).hex()}")
             print(f"Decoded length: {len(witness_program)} bytes")
     ```
   - Enregistrez et quittez (`Ctrl+O`, `Enter`, `Ctrl+X`).

4. **Exécuter le script de test Python** :
   - Exécutez le script pour confirmer que l'adresse Nito est correctement décodée :
     ```bash
     python3 /var/miningcore/test_bech32.py
     ```
   - Résultat attendu :
     ```
     HRP: nito
     WitVersion: 0
     Decoded (hex): 1543c36f34e58dd2a5153d4991f45932e9f7fbad
     Decoded length: 20 bytes
     ```

### Étape 3 : Intégrer le décodeur Bech32 personnalisé dans `BitcoinUtils.cs`
Après avoir réinstallé MiningCore, vous devez intégrer notre décodeur/encodeur Bech32 personnalisé dans `BitcoinUtils.cs` pour qu'il soit utilisé dans `BechSegwitAddressToDestination`. Voici les étapes :

1. **Localiser `BitcoinUtils.cs`** :
   - Ouvrez le fichier `BitcoinUtils.cs` dans le répertoire source de MiningCore :
     ```bash
     nano /var/miningcore/src/Miningcore/Blockchain/Bitcoin/BitcoinUtils.cs
     ```

2. **Ajouter les dépendances nécessaires** :
   - Assurez-vous que les directives `using` suivantes sont présentes au début du fichier. Si elles ne le sont pas, ajoutez-les :
     ```csharp
     using System.Text; // Pour StringBuilder
     using System; // Pour FormatException et types de base
     ```
   - Le fichier devrait déjà contenir :
     ```csharp
     using System.Diagnostics;
     using NBitcoin;
     using NBitcoin.DataEncoders;
     ```

3. **Remplacer `BechSegwitAddressToDestination` et ajouter le décodeur** :
   - Remplacez la méthode `BechSegwitAddressToDestination` par celle qui utilise notre décodeur personnalisé, et ajoutez les méthodes de décodage/encodage à la fin du fichier. Voici le contenu complet de `BitcoinUtils.cs` après modification :

  
   using System.Diagnostics;
   using NBitcoin;
   using NBitcoin.DataEncoders;
   using System.Text;
   using System;

   namespace Miningcore.Blockchain.Bitcoin
   {
       public static class BitcoinUtils
       {
           /// <summary>
           /// Bitcoin addresses are implemented using the Base58Check encoding of the hash of either:
           /// Pay-to-script-hash (p2sh): payload is: RIPEMD160(SHA256(redeemScript)) where redeemScript is a
           /// script the wallet knows how to spend; version byte = 0x05 (these addresses begin with the digit '3')
           /// Pay-to-pubkey-hash (p2pkh): payload is: RIPEMD160(SHA256(ECDSA_publicKey)) where
           /// ECDSA_publicKey is a public key the wallet knows the private key for; version byte = 0x00
           /// (these addresses begin with the digit '1')
           /// The resulting hash in both of these cases is always exactly 20 bytes.
           /// </summary>
           public static IDestination AddressToDestination(string address, Network expectedNetwork)
           {
               var decoded = Encoders.Base58Check.DecodeData(address);
               var networkVersionBytes = expectedNetwork.GetVersionBytes(Base58Type.PUBKEY_ADDRESS, true);
               decoded = decoded.Skip(networkVersionBytes.Length).ToArray();
               var result = new KeyId(decoded);

               return result;
           }

           public static IDestination BechSegwitAddressToDestination(string address, Network expectedNetwork)
           {
               try
               {
                   // Décoder l'adresse Bech32 avec notre décodeur personnalisé
                   var (witVersion, witnessProgram, hrp) = DecodeBech32(address);

                   // Vérifier la version du témoin (doit être 0 pour P2WPKH)
                   if (witVersion != 0)
                       throw new FormatException($"Version de témoin Bech32 non prise en charge : {witVersion}");

                   // Créer un WitKeyId avec les données réelles
                   var result = new WitKeyId(witnessProgram);

                   // Valider que l'adresse encodée correspond à l'entrée
                   var reencodedAddress = EncodeBech32(hrp, witVersion, witnessProgram);
                   if (reencodedAddress != address)
                       throw new FormatException("L'adresse encodée ne correspond pas à l'entrée");

                   return result;
               }
               catch (Exception ex)
               {
                   throw new FormatException($"Format Bech32 invalide : {ex.Message}");
               }
           }

           public static IDestination BCashAddressToDestination(string address, Network expectedNetwork)
           {
               var bcash = NBitcoin.Altcoins.BCash.Instance.GetNetwork(expectedNetwork.ChainName);
               var trashAddress = bcash.Parse<NBitcoin.Altcoins.BCash.BTrashPubKeyAddress>(address);
               return trashAddress.ScriptPubKey.GetDestinationAddress(bcash);
           }

           public static IDestination LitecoinAddressToDestination(string address, Network expectedNetwork)
           {
               var litecoin = NBitcoin.Altcoins.Litecoin.Instance.GetNetwork(expectedNetwork.ChainName);
               var encoder = litecoin.GetBech32Encoder(Bech32Type.WITNESS_PUBKEY_ADDRESS, true);

               var decoded = encoder.Decode(address, out var witVersion);
               var result = new WitKeyId(decoded);

               Debug.Assert(result.GetAddress(litecoin).ToString() == address);
               return result;
           }

           // Décodeur Bech32 personnalisé
           private static (byte witVersion, byte[] witnessProgram, string hrp) DecodeBech32(string address)
           {
               // Alphabet Bech32
               const string charset = "qpzry9x8gf2tvdw0s3jn54khce6mua7l";
               const char separator = '1';

               // Décomposer l'adresse en HRP et données
               var parts = address.Split(new[] { separator }, 2);
               if (parts.Length != 2)
                   throw new FormatException("Adresse Bech32 invalide : format incorrect");

               var hrp = parts[0]; // "nito", "bc1", etc.
               var dataPart = parts[1]; // Reste de l'adresse après "1"

               // Décoder les données (convertir les caractères Bech32 en valeurs bits 5)
               byte[] decodedData = new byte[dataPart.Length];
               for (int i = 0; i < dataPart.Length; i++)
               {
                   int value = charset.IndexOf(dataPart[i]);
                   if (value == -1)
                       throw new FormatException($"Caractère Bech32 invalide : {dataPart[i]}");
                   decodedData[i] = (byte)value;
               }

               // Extraire la version du témoin (premier octet)
               byte witVersion = decodedData[0];
               if (witVersion > 16)
                   throw new FormatException($"Version de témoin Bech32 non prise en charge : {witVersion}");

               // Ignorer les 6 derniers octets (checksum) et le premier octet (witVersion)
               byte[] dataWithoutChecksum = new byte[decodedData.Length - 6 - 1];
               Array.Copy(decodedData, 1, dataWithoutChecksum, 0, dataWithoutChecksum.Length);

               // Convertir les données (bits 5) en bits 8
               byte[] witnessProgram = ConvertBits(dataWithoutChecksum, 5, 8, false);

               // Vérifier la longueur du programme témoin (20 octets pour P2WPKH)
               if (witnessProgram.Length != 20)
                   throw new FormatException($"Programme témoin Bech32 invalide : longueur {witnessProgram.Length}, attendu 20 octets");

               return (witVersion, witnessProgram, hrp);
           }

           // Encodeur Bech32 personnalisé
           private static string EncodeBech32(string hrp, byte witVersion, byte[] witnessProgram)
           {
               // Alphabet Bech32
               const string charset = "qpzry9x8gf2tvdw0s3jn54khce6mua7l";

               // Convertir le programme témoin (bits 8) en bits 5
               var data = ConvertBits(witnessProgram, 8, 5, true);
               if (data == null)
                   throw new FormatException("Échec de la conversion du programme témoin en bits 5");

               // Ajouter la version du témoin au début
               var values = new byte[data.Length + 1];
               values[0] = witVersion;
               Array.Copy(data, 0, values, 1, data.Length);

               // Calculer le checksum Bech32
               var checksum = CreateBech32Checksum(hrp, values);
               var combined = new byte[values.Length + checksum.Length];
               Array.Copy(values, 0, combined, 0, values.Length);
               Array.Copy(checksum, 0, combined, values.Length, checksum.Length);

               // Encoder les données en caractères Bech32
               var sb = new StringBuilder();
               sb.Append(hrp);
               sb.Append('1');
               foreach (var value in combined)
               {
                   sb.Append(charset[value]);
               }

               return sb.ToString();
           }

           // Calculer le checksum Bech32
           private static byte[] CreateBech32Checksum(string hrp, byte[] values)
           {
               const int BECH32_CONST = 1;
               var hrpExpanded = ExpandHrp(hrp);
               var combined = new byte[hrpExpanded.Length + values.Length + 6];
               Array.Copy(hrpExpanded, 0, combined, 0, hrpExpanded.Length);
               Array.Copy(values, 0, combined, hrpExpanded.Length, values.Length);

               var polymod = Polymod(combined) ^ BECH32_CONST;
               var checksum = new byte[6];
               for (int i = 0; i < 6; i++)
               {
                   checksum[i] = (byte)((polymod >> (5 * (5 - i))) & 31);
               }

               return checksum;
           }

           // Étendre le HRP pour le calcul du checksum
           private static byte[] ExpandHrp(string hrp)
           {
               var result = new byte[2 * hrp.Length + 1];
               for (int i = 0; i < hrp.Length; i++)
               {
                   result[i] = (byte)(hrp[i] >> 5);
                   result[i + hrp.Length + 1] = (byte)(hrp[i] & 31);
               }
               return result;
           }

           // Calculer le polymod pour le checksum Bech32
           private static uint Polymod(byte[] values)
           {
               uint chk = 1;
               uint[] generator = { 0x3b6a57b2, 0x26508e6d, 0x1ea119fa, 0x3d4233dd, 0x2a1462b3 };
               foreach (var value in values)
               {
                   uint top = chk >> 25;
                   chk = (chk & 0x1ffffff) << 5 ^ value;
                   for (int i = 0; i < 5; i++)
                   {
                       if (((top >> i) & 1) != 0)
                           chk ^= generator[i];
                   }
               }
               return chk;
           }

           // Méthode utilitaire pour convertir les bits 5 en bits 8
           private static byte[] ConvertBits(byte[] data, int fromBits, int toBits, bool pad = true)
           {
               int acc = 0;
               int bits = 0;
               var result = new System.Collections.Generic.List<byte>();
               int maxv = (1 << toBits) - 1;

               foreach (var value in data)
               {
                   if (value < 0 || value >> fromBits != 0)
                       throw new FormatException("Valeur Bech32 invalide");

                   acc = (acc << fromBits) | value;
                   bits += fromBits;

                   while (bits >= toBits)
                   {
                       bits -= toBits;
                       result.Add((byte)((acc >> bits) & maxv));
                   }
               }

               if (pad)
               {
                   if (bits > 0)
                       result.Add((byte)((acc << (toBits - bits)) & maxv));
               }
               else if (bits >= fromBits || ((acc << (toBits - bits)) & maxv) != 0)
               {
                   throw new FormatException("Conversion Bech32 invalide : bits restants non nuls");
               }

               return result.ToArray();
           }
       }
   }
   

4. **Enregistrer et quitter** :
   - Enregistrez les modifications dans `nano` (`Ctrl+O`, `Enter`, `Ctrl+X`).

### Étape 4 : Recompiler MiningCore
- Recompilez MiningCore pour prendre en compte les modifications :
  ```bash
  cd /var/miningcore/src
  sudo dotnet build -c Release
  ```
- Assurez-vous que la compilation réussit.

### Étape 5 : Redémarrer MiningCore
- Redémarrez le service MiningCore pour appliquer les modifications :
  ```bash
  sudo systemctl restart miningcore
  ```

### Résumé des dépendances
#### Pour `BitcoinUtils.cs` (C#)
- `System.Text` : Pour `StringBuilder` utilisé dans `EncodeBech32`.
- `System` : Pour `FormatException` et les types de base comme `byte`.

#### Pour les tests (Python, si nécessaire)
- **Python 3** : Pour exécuter le script de test.
- **Bibliothèque Python `bech32`** : Pour valider le décodage Bech32.

### Conclusion
- **Modification** : Nous avons remplacé NBitcoin pour le décodage des adresses Bech32 dans `BechSegwitAddressToDestination` par notre décodeur personnalisé.
- **Dépendances** : Ajout de `System.Text` et `System` au début de `BitcoinUtils.cs`, et Python 3 avec la bibliothèque `bech32` pour les tests.
- **Prochaines étapes** : Recompilez et redémarrez MiningCore, puis surveillez les logs pour confirmer que tout fonctionne.

