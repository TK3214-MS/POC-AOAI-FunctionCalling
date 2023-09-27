# アーキテクチャー
今回のアーキテクチャーは以下の通りです。

![00](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/a76dd40d-256b-47b1-9048-19b840be6212)

簡単な流れは以下の通りです。

1. Power Apps より音声で質問を入力する。
1. 入力された音声データが base64 形式でPower Automate フローに渡される。
1. 音声データを Azure Function が WMV フォーマットに変換する。
1. 音声データが Azure Speech Service によりテキストに変換される。
1. 変換後テキストを元に Azure Open AI Service が社内ドキュメント検索か通常の応答かを判断する。
1. 社内ドキュメント検索が必要と判断された場合、事前定義された JSON スキーマを元に Graph API での Microsoft Search を実行する。

# 構成手順
以下を構成、実装する事で動作確認用環境の構成が可能です。

## 事前準備
### Azure Open AI リソースの作成
1. [手順ページ](https://learn.microsoft.com/ja-jp/azure/ai-services/openai/how-to/create-resource?pivots=web-portal)に則り、Azure Open AI Service リソースを作成します。

1. [手順ページ](https://learn.microsoft.com/ja-jp/azure/ai-services/openai/how-to/create-resource?pivots=web-portal#deploy-a-model)に則り、モデルを作成します。

    ※本サンプルでは gpt-35-turbo-16k を利用しました。

### SharePoint Online ドキュメントライブラリの準備
1. 検索対象としたいドキュメントをアップロードします。

### Azure AI Speech Service リソースの作成
Speech Service リソース (無料または有料レベル) を Azure アカウントに追加するには:

1. お使いの Microsoft アカウントを使用して [Azure portal](https://portal.azure.com/) にサインインします。

1. ポータルの左上にある **[Create a resource]\(リソースの作成\)** を選択します。 **[リソースの作成]** が表示されない場合は、画面左上の折りたたまれたメニューを選択することで、いつでも見つけることができます。

1. **新規** ウィンドウで、検索ボックスに「speech」と入力し、Enter キーを押します。

1. 検索結果で、 **[Speech]** を選択します。
   
   :::image type="content" source="media/index/speech-search.png" alt-text="Azure portal で Speech リソースを作成します。":::

1. **[作成]** を選択して、次のことを行います。

   - 新しいリソースに一意の名前を指定します。 この名前は、同じサービスに関連付けられた複数のサブスクリプションを区別するのに役立ちます。
   - 新しいリソースが関連付けられている Azure サブスクリプションを選択して、料金の課金方法を決定します。 Azure portal で [Azure サブスクリプションを作成する方法](../../cost-management-billing/manage/create-subscription.md#create-a-subscription-in-the-azure-portal)の概要はこちらにあります。
   - リソースが使用される[リージョン](regions.md)を選択します。 Azure は、世界中のさまざまな地域で一般的に利用できるグローバル クラウド プラットフォームです。 パフォーマンスを最適にするには、ユーザーまたはアプリケーションが実行されている場所に最も近いリージョンを選択します。 音声サービスの可用性は、リージョンによって異なります。 サポートされているリージョンにリソースが作成されていることを確認してください。 [音声サービスがサポートされているリージョン](./regions.md#speech-to-text-text-to-speech-and-translation)に関するページを参照してください。
   - 無料 (F0) または有料 (S0) の価格レベルのどちらかを選択します。 各レベルの価格と使用量クォータの完全な情報については、 **[価格の詳細を表示]** を選択するか、![音声サービスの価格](https://azure.microsoft.com/pricing/details/cognitive-services/speech-services/)に関するページを参照してください。 リソースの制限については、「[Azure Cognitive Services の制限](../../azure-resource-manager/management/azure-subscription-service-limits.md#azure-cognitive-services-limits)」を参照してください。
   - この Speech サブスクリプションの新しいリソース グループを作成するか、既存のリソース グループにサブスクリプションを割り当てます。 リソース グループは、さまざまな Azure サブスクリプションを整理しておくのに役立ちます。
   - **［作成］** を選択します これでデプロイの概要に移動し、デプロイの進行状況を示すメッセージが表示されます。  

新しい音声リソースを展開するまでに少し時間がかかります。 

### キーと場所/リージョンを見つける

完成したデプロイのキーと場所/リージョンを見つけるには、次の手順に従います。

1. お使いの Microsoft アカウントを使用して [Azure portal](https://portal.azure.com/) にサインインします。

2. **[すべてのリソース]** を選択し、Cognitive Services リソースの名前を選択します。

3. 左ペインの **[リソース管理]** から **[Keys and Endpoint]\(キーとエンドポイント\)** を選択します。

各サブスクリプションには 2 つのキーがあります。アプリケーションでどちらのキーを使用しても構いません。 キーをコード エディターやその他の場所にコピーして貼り付けるには、各キーの横にあるコピー ボタンを選択し、ウィンドウを切り替えてクリップボードの内容を目的の場所に貼り付けます。

さらに、SDK 呼び出しのリージョン ID (、 など) である  値をコピーします。 SDK 呼び出しのリージョン ID (`westus`、`westeurope` など) です。

> [!IMPORTANT]
> これらのサブスクリプション キーは、Cognitive Service API にアクセスするために使用されます。 キーを共有しないでください。 Azure Key Vault を使用するなどして、安全に保管してください。 これらのキーを定期的に再生成することもお勧めします。 API 呼び出しを行うために必要なキーは 1 つだけです。 最初のキーを再生成するときに、2 番目のキーを使用してサービスに継続的にアクセスすることができます。

### Function App リソースの作成
1. [手順ページ](https://learn.microsoft.com/ja-jp/azure/azure-functions/functions-create-function-app-portal#create-a-function-app)に則り、Function App リソースを作成します。

1. 本サンプルでは以下設定でリソースを作成しました。

    | 設定名 | 設定値 |
    | ---- | ---- |
    | Runtime stack | .NET |
    | Version | 6 (LTS) |
    | Operating System | Windows |
    | Hosting option | Consumption |

    ※その他項目は任意で設定して下さい。

## 構成
### Function App の構成
1. 作成した Function App リソースから新しい関数を以下設定値で作成します。

    ![01](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/15f7c542-7430-4c0d-a2bb-302abbfe6d95)

1. 作成した関数の"Code+Test"を選択し、[run.csx](/run.csx)、[function.json](/function.json)の内容を貼り付けます。

1. 変更を保存し、"Get function URL"より Power Automate より HTTP 要求を行うターゲット URL をコピーしておきます。

    ![02](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/5e8a7cf6-de6a-4cc0-b1f9-0fc006d2d71e)

1. Function App リソースのトップ画面に戻り、"Advanced Tool" から Kudu を開きます。

1. "Debug Console" に "CMD" を選択し、以下パスまで移動します。

    C:\home\site\wwwroot\

1. "ConvertAudioFormatUsingFFmpeg" ディレクトリを新規作成し、![こちら](https://ffmpeg.org/download.html#build-windows)よりダウンロードした Windows 向け FFmpeg 実行ファイルをアップロードします。

    - ffmpeg.exe
    - ffplay.exe
    - ffprobe.exe

### Power Apps の構成
以下手順を構成頂くと以下アプリが完成します。

![03](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/ac590631-aaf2-45a9-95e8-ca0a272b4850)

| No | コントロール種別 | コントロール名 |
| ---- | ---- | ---- |
| ① | マイク | Microphone1 |
| ② | ボタン | Button1 |
| ③ | テキストラベル | AudioJSON |
| ④ | テキストラベル | TextOutput |
| ⑤ | オーディオ | Audio1 |

1. マイクコントロールへのプロパティ追加

    | プロパティ名 | 値 |
    | ---- | ---- |
    | OnStop | ClearCollect(AudioCollection,Microphone1.Audio);<br>Set(JSONValue,JSON(AudioCollection,JSONFormat.IncludeBinaryData)); |

1. ボタンコントロールへのプロパティ追加

    | プロパティ名 | 値 |
    | ---- | ---- |
    | OnSelect | Set(DesiredOutput,'POC-Convert-Speech2Text'.Run(JSON(First(AudioCollection),JSONFormat.IncludeBinaryData)));<br>Set(TTSAudio,DesiredOutput.audio);<br>Set(VisibleClip, true);<br>Set(playAudio,true);

1. テキストラベルコントロール(AudioJSON)へのプロパティ追加

    | プロパティ名 | 値 |
    | ---- | ---- |
    | Text | Text(JSONValue) |

1. テキストラベルコントロール(TextOutput)へのプロパティ追加

    | プロパティ名 | 値 |
    | ---- | ---- |
    | Text | DesiredOutput.text |

    > [!NOTE]<br>
    > 複数のエラーが表示されますが、Power Automate フローを関連付けると解消されます。

### Power Automate の構成
以下手順を構成頂くと以下フローが完成します。

![04](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/6d6f23ce-e4c9-40aa-9082-886a815702bf)

1. Power Apps トリガーの構成

![05](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/8217eb14-8229-473f-82ff-b2c5b7da2ebd)

1. 変数の初期化 (Initialize AOAI URL) アクションの構成

    ![06](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/8a5b3cb1-4777-44fa-b91a-c91fe1f5490d)

    | 名前 | 種類 | 値 |
    | ---- | ---- | ---- |
    | AOAI-URL | 文字列 | https://**[AOAI ENDPOINT]**.openai.azure.com/openai/deployments/**[MODEL NAME]**]/chat/completions?api-version=2023-07-01-preview |

1. 変数の初期化 (Initialize AOAI Key) アクションの構成

    ![07](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/58a2c82e-6f61-4ae3-bd95-298db29011cd)

    | 名前 | 種類 | 値 |
    | ---- | ---- | ---- |
    | AOAI-Key | 文字列 | **[AOAI KEY]** |

1. 作成 (Compose PowerApps Input) アクションの構成

    ![08](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/30459691-af09-4ac8-9170-bbcfa03e3387)

    | 入力 |
    | ---- |
    | triggerBody()['ComposePowerAppsInput_入力'] |

    ※式(Power Fx)として入力

1. JSON の解析 (Parse Power Apps Input) アクションの構成

    ![09](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/dcc807c3-398f-47bf-8dac-67453c4a79b8)

    | コンテンツ | スキーマ |
    | ---- | ---- |
    | outputs('Compose_PowerApps_Input') | {<br>    "type": "object",<br>    "properties": {<br>        "Value": {<br>            "type": "string"<br>        }<br>    }<br>}

1. HTTP (Call Media Format Conversion) アクションの構成

    ![10](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/f1728c2e-a900-4107-ba98-7ad0c63fef1e)

    | 方法 | URI | 本文 |
    | ---- | ---- | ---- |
    | POST | **[Function App URL]** | body('Parse_Power_Apps_Input')?['Value'] |

1. HTTP (Call Speech2Text Service) アクションの構成

    ![11](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/d72e4f0d-64a8-4a86-8855-090942c3d8b6)

    | 方法 | URI | ヘッダー | 本文 |
    | ---- | ---- | ---- | ---- |
    | POST | https://japaneast.stt.speech.microsoft.com/speech/recognition/conversation/cognitiveservices/v1?language=ja-JP&format=detailed | Ocp-Apim-Subscription-Key：**[Speech Service Key]**<br>Content-type：audio/wav | body('Call_Media_Format_Conversion') |

1. JSON の解析 (Parse Speech2Text Output) アクションの構成

    ![12](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/3b9be6d8-73dd-4c5a-9f22-bddaf4239a04)

    | コンテンツ | スキーマ |
    | ---- | ---- |
    | body('Call_Speech2Text_Service') | {<br>    "type": "object",<br>    "properties": {<br>        "RecognitionStatus": {<br>            "type": "string"<br>        },<br>        "Offset": {<br>            "type": "integer"<br>        },<br>        "Duration": {<br>            "type": "integer"<br>        },<br>        "NBest": {<br>            "type": "array",<br>            "items": {<br>                "type": "object",<br>                "properties": {<br>                    "Confidence": {<br>                        "type": "number"<br>                    },<br>                    "Lexical": {<br>                        "type": "string"<br>                    },<br>                    "ITN": {<br>                        "type": "string"<br>                    },<br>                    "MaskedITN": {<br>                        "type": "string"<br>                    },<br>                    "Display": {<br>                        "type": "string"<br>                    }<br>                },<br>                "required": [<br>                    "Confidence",<br>                    "Lexical",<br>                    "ITN",<br>                    "MaskedITN",<br>                    "Display"<br>                ]<br>            }<br>        },<br>        "DisplayText": {<br>            "type": "string"<br>        }<br>    }<br>} |

1. HTTP (Call AOAI for Output) アクションの構成

    ![13](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/919c1ce7-bc18-4af3-b9d4-adf6425e5cd7)

    | 方法 | URI | ヘッダー | 本文 |
    | ---- | ---- | ---- | ---- |
    | POST | variables('AOAI-URL') | api-key：variables('AOAI-Key')<br>Content-type：application/json | 以下を参照 |

    ```json
    {
        "frequency_penalty": 0,
        "max_tokens": 3000,
        "messages": [
            {
                "content": "You are AI assisstant.",
                "role": "system"
            },
            {
                "content": "@{body('Parse_Speech2Text_Output')?['DisplayText']}",
                "role": "user"
            }
        ],
        "functions": [
            {
                "name": "microsoft_search_api",
                "description": "手順やルールのドキュメントに関する質問をされるとMicrosoft Graph Search APIを用いて検索をします。",
                "parameters": {
                    "type": "object",
                    "properties": {
                        "requests": {
                        "type": "object",
                    "properties": {
                        "entityTypes": {
                        "type": "array",
                        "description": "検索の対象にするリソースの種類。不明な場合はすべての項目を含めます。",
                        "items": {
                            "type": "string",
                            "enum": [
                                "site",
                                "list",
                                "listItem",
                                "drive",
                                "driveItem",
                                "message",
                                "event"
                                ]
                            }
                        },
                        "query": {
                            "type": "object",
                            "properties": {
                            "queryString": {
                            "type": "string",
                            "description": "検索キーワード。半角スペース区切り。"
                            }
                        }
                    }
                }
            }
        }
    }
    }
    ],
    "presence_penalty": 0,
    "stop": null,
    "temperature": 0
    }

    ```

1. JSON の解析 (Parse AOAI Output) アクションの構成

    ![14](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/1e07e777-3f7f-4f72-8531-9b5998d69c0d)

    | コンテンツ | スキーマ |
    | ---- | ---- |
    | body('Call_AOAI_for_Output') | 以下を参照 |

    ```json
        {
        "type": "object",
        "properties": {
            "id": {
                "type": "string"
            },
            "object": {
                "type": "string"
            },
            "created": {
                "type": "integer"
            },
            "model": {
                "type": "string"
            },
            "prompt_filter_results": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "prompt_index": {
                            "type": "integer"
                        },
                        "content_filter_results": {
                            "type": "object",
                            "properties": {
                                "hate": {
                                    "type": "object",
                                    "properties": {
                                        "filtered": {
                                            "type": "boolean"
                                        },
                                        "severity": {
                                            "type": "string"
                                        }
                                    }
                                },
                                "self_harm": {
                                    "type": "object",
                                    "properties": {
                                        "filtered": {
                                            "type": "boolean"
                                        },
                                        "severity": {
                                            "type": "string"
                                        }
                                    }
                                },
                                "sexual": {
                                    "type": "object",
                                    "properties": {
                                        "filtered": {
                                            "type": "boolean"
                                        },
                                        "severity": {
                                            "type": "string"
                                        }
                                    }
                                },
                                "violence": {
                                    "type": "object",
                                    "properties": {
                                        "filtered": {
                                            "type": "boolean"
                                        },
                                        "severity": {
                                            "type": "string"
                                        }
                                    }
                                }
                            }
                        }
                    },
                    "required": [
                        "prompt_index",
                        "content_filter_results"
                    ]
                }
            },
            "choices": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "index": {
                            "type": "integer"
                        },
                        "finish_reason": {
                            "type": "string"
                        },
                        "message": {
                            "type": "object",
                            "properties": {
                                "role": {
                                    "type": "string"
                                },
                                "function_call": {
                                    "type": "object",
                                    "properties": {
                                        "name": {
                                            "type": "string"
                                        },
                                        "arguments": {
                                            "type": "string"
                                        }
                                    }
                                }
                            }
                        },
                        "content_filter_results": {
                            "type": "object",
                            "properties": {}
                        }
                    },
                    "required": [
                        "index",
                        "finish_reason",
                        "message",
                        "content_filter_results"
                    ]
                }
            },
            "usage": {
                "type": "object",
                "properties": {
                    "completion_tokens": {
                        "type": "integer"
                    },
                    "prompt_tokens": {
                        "type": "integer"
                    },
                    "total_tokens": {
                        "type": "integer"
                    }
                }
            }
        }
    }

    ```

1. 作成 (Pick first argument) アクションの構成

    ![15](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/a94c3424-7259-45c8-9f98-bce8577ff069)

    | 入力 |
    | ---- |
    | first(body('Parse_AOAI_Output')?['choices']) |

    ※式(Power Fx)として入力

1. JSON の解析 (Parse argument item) アクションの構成

    ![16](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/869e64c4-c0aa-422d-bfc9-755385884fed)

    | コンテンツ | スキーマ |
    | ---- | ---- |
    | outputs('Pick_first_argument') | 以下を参照 |

    ```json
    {
        "type": "object",
        "properties": {
            "index": {
                "type": "integer"
            },
            "finish_reason": {
                "type": "string"
            },
            "message": {
                "type": "object",
                "properties": {
                    "role": {
                        "type": "string"
                    },
                    "function_call": {
                        "type": "object",
                        "properties": {
                            "name": {
                                "type": "string"
                            },
                            "arguments": {
                                "type": "string"
                            }
                        }
                    }
                }
            },
            "content_filter_results": {
                "type": "object",
                "properties": {}
            }
        }
    }

    ```

1. 変数の初期化 (Initialize Argument) アクションの構成

    ![17](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/32479249-fd01-4652-af5f-355abd2117fc)

    | 名前 | 種類 | 値 |
    | ---- | ---- | ---- |
    | Argument | オブジェクト | [空白] |

1. 変数の設定 (Set Argument) アクションの構成

    ![18](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/6f78628e-553e-478d-8206-f0db9ee738fb)

    | 名前 | 値 |
    | ---- | ---- |
    | Argument | {<br>  "requests": [<br>    {<br>      "entityTypes": [<br>        "listItem"<br>      ],<br>      "query": {<br>        "queryString": "@{body('Parse_argument_item')?['message']?['function_call']?['arguments']?['queryString']}"<br>      }<br>    }<br>  ]<br>} |

1. 条件コントロールの構成

    ![19](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/f8cdfada-c001-4225-a47c-12237fd6d047)

    | 条件 | 式 | 値 |
    | ---- | ---- | ---- |
    | body('Parse_argument_item')?['finish_reason'] | 次の値に等しい | function_call |

### はいの場合

1. HTTP with Azure AD (Request Search via Graph API) の構成

    ![20](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/3acd53e4-6dd4-433e-8e86-2deb495ba965)

    | 方法 | 要求の URL | 要求の本文 |
    | ---- | ---- | ---- |
    | POST | https://graph.microsoft.com/v1.0/search/query | variables('Argument') |

1. HTTP (Validate Answer via AOAI) アクションの構成

    ![21](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/98f8a6a2-c62f-4c02-a230-cec6e9c260c4)

    | 方法 | URI | ヘッダー | 本文 |
    | ---- | ---- | ---- | ---- |
    | POST | variables('AOAI-URL') | api-key：variables('AOAI-Key')<br>Content-type：application/json | 以下を参照 |

    ```json
    {
        "frequency_penalty": 0,
        "max_tokens": 3000,
        "messages": [
        {
            "content": "You are AI assisstant.",
            "role": "system"
        },
        {
            "content": "@{body('Parse_Speech2Text_Output')?['DisplayText']}",
            "role": "user"
        },
        {
            "content": "@{body('Request_Search_via_Graph_API')}",
            "role": "function",
            "name": "microsoft_search_api"
        }
        ],
        "functions": [
        {
            "name": "microsoft_search_api",
            "description": "手順やルールのドキュメントに関する質問をされるとMicrosoft Graph Search APIを用いて検索をします。",
            "parameters": {
            "type": "object",
            "properties": {
                "requests": {
                "type": "object",
                "properties": {
                    "entityTypes": {
                    "type": "array",
                    "description": "検索の対象にするリソースの種類。不明な場合はすべての項目を含めます。",
                    "items": {
                        "type": "string",
                        "enum": [
                        "site",
                        "list",
                        "listItem",
                        "drive",
                        "driveItem",
                        "message",
                        "event"
                        ]
                    }
                    },
                    "query": {
                    "type": "object",
                    "properties": {
                        "queryString": {
                        "type": "string",
                        "description": "検索キーワード。半角スペース区切り。"
                        }
                    }
                    }
                }
                }
            }
            }
        }
        ],
        "presence_penalty": 0,
        "stop": null,
        "temperature": 0
    }
    ```

1. JSON の解析 (Parse Search API Output) アクションの構成

    ![22](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/b613cd29-2dd8-4cdb-b28f-ffe9d6cf32c2)

    | コンテンツ | スキーマ |
    | ---- | ---- |
    | body('Validate_Answer_via_AOAI') | 以下を参照 |

    ```json
    {
        "type": "object",
        "properties": {
            "id": {
                "type": "string"
            },
            "object": {
                "type": "string"
            },
            "created": {
                "type": "integer"
            },
            "model": {
                "type": "string"
            },
            "prompt_filter_results": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "prompt_index": {
                            "type": "integer"
                        },
                        "content_filter_results": {
                            "type": "object",
                            "properties": {
                                "hate": {
                                    "type": "object",
                                    "properties": {
                                        "filtered": {
                                            "type": "boolean"
                                        },
                                        "severity": {
                                            "type": "string"
                                        }
                                    }
                                },
                                "self_harm": {
                                    "type": "object",
                                    "properties": {
                                        "filtered": {
                                            "type": "boolean"
                                        },
                                        "severity": {
                                            "type": "string"
                                        }
                                    }
                                },
                                "sexual": {
                                    "type": "object",
                                    "properties": {
                                        "filtered": {
                                            "type": "boolean"
                                        },
                                        "severity": {
                                            "type": "string"
                                        }
                                    }
                                },
                                "violence": {
                                    "type": "object",
                                    "properties": {
                                        "filtered": {
                                            "type": "boolean"
                                        },
                                        "severity": {
                                            "type": "string"
                                        }
                                    }
                                }
                            }
                        }
                    },
                    "required": [
                        "prompt_index",
                        "content_filter_results"
                    ]
                }
            },
            "choices": {
                "type": "array",
                "items": {
                    "type": "object",
                    "properties": {
                        "index": {
                            "type": "integer"
                        },
                        "finish_reason": {
                            "type": "string"
                        },
                        "message": {
                            "type": "object",
                            "properties": {
                                "role": {
                                    "type": "string"
                                },
                                "content": {
                                    "type": "string"
                                }
                            }
                        },
                        "content_filter_results": {
                            "type": "object",
                            "properties": {
                                "hate": {
                                    "type": "object",
                                    "properties": {
                                        "filtered": {
                                            "type": "boolean"
                                        },
                                        "severity": {
                                            "type": "string"
                                        }
                                    }
                                },
                                "self_harm": {
                                    "type": "object",
                                    "properties": {
                                        "filtered": {
                                            "type": "boolean"
                                        },
                                        "severity": {
                                            "type": "string"
                                        }
                                    }
                                },
                                "sexual": {
                                    "type": "object",
                                    "properties": {
                                        "filtered": {
                                            "type": "boolean"
                                        },
                                        "severity": {
                                            "type": "string"
                                        }
                                    }
                                },
                                "violence": {
                                    "type": "object",
                                    "properties": {
                                        "filtered": {
                                            "type": "boolean"
                                        },
                                        "severity": {
                                            "type": "string"
                                        }
                                    }
                                }
                            }
                        }
                    },
                    "required": [
                        "index",
                        "finish_reason",
                        "message",
                        "content_filter_results"
                    ]
                }
            },
            "usage": {
                "type": "object",
                "properties": {
                    "completion_tokens": {
                        "type": "integer"
                    },
                    "prompt_tokens": {
                        "type": "integer"
                    },
                    "total_tokens": {
                        "type": "integer"
                    }
                }
            }
        }
    }
    ```

1. 作成 (Pick first content) アクションの構成

    ![23](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/9bbf3725-3940-40ed-91f0-8f64766948f1)

    | 入力 |
    | ---- |
    | first(body('Parse_Search_API_Output')?['choices'])

1. JSON の解析 (Parse Content) アクションの構成

    ![24](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/2b63fa47-b78c-4067-a44c-c6123ec33230)

    | コンテンツ | スキーマ |
    | ---- | ---- |
    | outputs('Pick_first_content') | 以下を参照 |

    ```json
    {
        "type": "object",
        "properties": {
            "index": {
                "type": "integer"
            },
            "finish_reason": {
                "type": "string"
            },
            "message": {
                "type": "object",
                "properties": {
                    "role": {
                        "type": "string"
                    },
                    "content": {
                        "type": "string"
                    }
                }
            },
            "content_filter_results": {
                "type": "object",
                "properties": {
                    "hate": {
                        "type": "object",
                        "properties": {
                            "filtered": {
                                "type": "boolean"
                            },
                            "severity": {
                                "type": "string"
                            }
                        }
                    },
                    "self_harm": {
                        "type": "object",
                        "properties": {
                            "filtered": {
                                "type": "boolean"
                            },
                            "severity": {
                                "type": "string"
                            }
                        }
                    },
                    "sexual": {
                        "type": "object",
                        "properties": {
                            "filtered": {
                                "type": "boolean"
                            },
                            "severity": {
                                "type": "string"
                            }
                        }
                    },
                    "violence": {
                        "type": "object",
                        "properties": {
                            "filtered": {
                                "type": "boolean"
                            },
                            "severity": {
                                "type": "string"
                            }
                        }
                    }
                }
            }
        }
    }
    ```

1. HTTP (Call Text2Speech Service) アクションの構成

    ![25](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/5aa39d3d-11ea-461a-a2ec-4c988a819dea)

    | 方法 | URI | ヘッダー | 本文 |
    | --- | --- | --- | --- |
    | POST | https://japaneast.tts.speech.microsoft.com/cognitiveservices/v1 | Ocp-Apim-Subscription-Key：Sppech Service KEY<br>Content-type：application/ssml+xml<br>X-microsoft-OutputFormat：audio-16khz-128kbitrate-mono-mp3 | 以下を参照 |

    ```xml
    <speak version='1.0' xml:lang='ja-JP'><voice xml:lang='ja-JP' xml:gender='Female'
        name='ja-JP-NanamiNeural'>
            @{body('Parse_Content')?['message']?['content']}
    </voice></speak>
    ```

1. 作成 (Create Audio Media) アクションの構成

    ![26](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/04d53901-a473-4f64-9749-4d40454ca66e)

    | 入力 |
    | ---- |
    | data:audio/mpeg;base64, body('Call_Text2Speech_Service')?['$content'] |

1. PowerApp または Flow に応答する アクションの構成

    ![27](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/99f79fdd-3851-48a3-8321-fc8c4bb97612)

    | 形式 | 名前 | 値 |
    |---- | ---- | ---- |
    | 文字列 | text | body('Parse_Content')?['message']?['content'] |
    | 文字列 | text | outputs('Create_Audio_Media') |

#### いいえの場合
1. HTTP (Call Text2Speech for chatGPT) アクションの構成

    ![28](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/52c9d8c0-7f75-49a4-95c7-ed6a32d298aa)

    | 方法 | URI | ヘッダー | 本文 |
    | ---- | ---- | ---- | ---- |
    | POST | https://japaneast.tts.speech.microsoft.com/cognitiveservices/v1 | Ocp-Apim-Subscription-Key：Speech Service KEY<br>Content-type：application/ssml+xml<br>X-microsoft-OutputFormat：audio-16khz-128kbitrate-mono-mp3 | 以下を参照 |

    ```xml
    <speak version='1.0' xml:lang='ja-JP'><voice xml:lang='ja-JP' xml:gender='Male'
        name='ja-JP-DaichiNeural'>
            @{body('Parse_argument_item')?['message']?['content']}
    </voice></speak>
    ```

1. 作成 (Create Audio for chatGPT) アクションの構成

    ![29](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/7f86a7ea-3c56-4ea3-937e-f85e64c22135)

    | 入力 |
    | ---- |
    | data:audio/mpeg;base64, body('Call_Text2Speech_for_chatGPT')?['$content'] |

1. PowerApp または Flow に応答する アクションの構成

    ![30](https://github.com/TK3214-MS/POC-AUTH/assets/89323076/c46a8022-eb4c-4946-9e2c-399c8d02eb28)

    | 形式 | 名前 | 値 |
    |---- | ---- | ---- |
    | 文字列 | text | body('Parse_argumen_item')?['message']?['content'] |
    | 文字列 | text | outputs('Create_Audio_for_chatGPT') |

## 動作確認
1. ![こちら](https://learn.microsoft.com/ja-jp/power-apps/maker/canvas-apps/using-logic-flows#add-a-flow-to-an-app)を参考に Power Apps 上で作成した Power Automate フローを関連付けます。

1. Power Apps から質問を投げかけます。

1. 検索対象が社内ドキュメントであれば女性の声、その他であれば男性の声で応答があれば動作確認成功です。