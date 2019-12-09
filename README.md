# Google Pay SDK

## Build

Para compilar o SDK, basta ir até o terminal, no diretório raiz do projeto, e digitar:
 
 ```
 ./gradlew :googlepay:clean :googlepay:assembleRelease
 ```

O arquivo AAR (`braspag-google-pay-release.aar`) estará disponível no diretório `./googlepay/build/outputs/aar`


## Gerando uma nova versão

Para incrementar a versão da biblioteca, edite o arquivo `build.gradle` localizado na pasta `googlepay`:

```
ext {
    libraryVersion = '1.0.0'
}
```

Altere a variável `libraryVersion` para refletir a nova versão.

Edite também no mesmo arquivo o campo `versionCode` (basta incrementar o valor).

Com isso, você pode publicar essa nova versão no JCenter
 
## Publicação no JCenter (Bintray)

Será necessário criar o arquivo `keystore.gradle` na pasta `googlepay`.
Neste arquivo ficam armazenados o usuário do Bintray e a API key:

```
ext {
    bintray_user = 'usuario_bintray'
    bintray_key = 'exxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
}
```

Estes dados são sensíveis e não devem ser versionados!

Para publicar manualmente uma nova versão no JCenter, basta executar o seguinte comando:

```
./gradlew :googlepay:bintrayUpload
```

## Continuous Integration com Travis

O projeto está configurado para integração contínua no Travis. As seguintes variáveis de ambiente  - `BINTRAY_USER` e `BINTRAY_KEY` - devem estar definidas no Travis.

A publicação do AAR no Bintray só será feita quando o branch for o ***master***.

## Utilização em app clientes

Primeiramente, deve ser alterado o `app/build.gradle` do cliente para adicionar as novas dependências:

```
implementation 'com.google.android.gms:play-services-wallet:18.0.0'
implementation 'br.com.braspag:braspag-google-pay:1.0.0'
```

Para utilizar o SDK, é necessário alterar o ***AndroidManifest.xml*** para incluir o seguinte metadado:

```
<meta-data
    android:name="com.google.android.gms.wallet.api.enabled"
    android:value="true" />
```

O ***AndroidManifest.xml*** do app fica assim:

```
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="br.com.braspag.googlepay.sample">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">
        <activity android:name=".MainActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />

                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <meta-data
            android:name="com.google.android.gms.wallet.api.enabled"
            android:value="true" />

    </application>

</manifest>
```

Para utilizar o SDK, é necessário importar os pacotes:

```
import br.com.braspag.googlepay.BraspagGooglePay
import br.com.braspag.googlepay.Environment
import br.com.braspag.googlepay.TransactionResult
```

Para instanciar o SDK:

```
sdk = BraspagGooglePay(
    merchantId = MERCHANT_ID,
    merchantName = "Lojão das Fábricas",
    environment = Environment.SANDBOX,
    activity = this,
    dataRequestCode = REQUEST_CODE
)
```

Atenção especial para a constante `REQUEST_CODE`
 
```
const val REQUEST_CODE = 666
```

Feito isso, é necessário verificar se o Google Pay está disponível no device do usuário.
Isso é feito através do método *`isGooglePayAvailable`*

Com ele é possível habilitar (ou exibir) o botão de pagamento do Google Pay.

```
sdk.isGooglePayAvailable {
    buttonGooglePay.isEnabled = it
}
```

Após isso, é possível fazer uma transação:

```
buttonGooglePay.setOnClickListener {
    sdk.makeTransaction(100.00)  // preço total em reais
}
```

O método *`makeTransaction`* irá exibir uma activity do `Google Pay`.

Após isso, é necessário tratar o retorno através do método `onActivityResult`:

```
override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
    super.onActivityResult(requestCode, resultCode, data)

    when (requestCode) {

        REQUEST_CODE -> {
            when (resultCode) {

                TransactionResult.SUCCESS.value ->
                    data?.let { intent ->
                        val paymentInfo = sdk.getDataFromIntent(intent)

                        Log.i(TAG, "paymentInfo: $paymentInfo")

                        //  continue authorization flow
                    }

                TransactionResult.USER_CANCELED.value -> {
                    // Nothing to do here normally - the user simply cancelled without selecting
                    // a payment method.

                    Toast.makeText(this, "Usuário cancelou pagamento!", Toast.LENGTH_LONG)
                        .show()
                }

                TransactionResult.ERROR.value -> {
                    data?.let {
                        val errorCode = sdk.getStatusFromIntent(it)

                        //  display error to user
                        Toast.makeText(this, "Erro na transação: $errorCode", Toast.LENGTH_LONG)
                            .show()
                    }
                }
            }

            buttonGooglePay.isEnabled = true
        }
    }
}
```

Para funcionar de acordo, o SDK deve ser utilizado em ambientes (emulador ou dispositivo real) onde exista uma conta de usuário Google ativa.

Caso contrário, na chamada do método *`makeTransaction`*, uma `dialogBox` como a seguir será exibida. O retorno nesse cenário será `TransactionResult.USER_CANCELED`.

![Conta Google não definida](images/no-account.png?raw=true)
