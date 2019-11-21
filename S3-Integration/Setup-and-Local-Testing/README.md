# Set up S3 Storage

```javascript
amplify add storage
```

## Interacting with Storage From React

1. Install the dependencies

```
yarn add aws-amplify aws-amplify-react
```

2. Add the React code

```javascript
import React, { useState } from "react"
import { withAuthenticator } from "aws-amplify-react"
import { Storage } from "aws-amplify"

import BuyLayout from "../../components/swap-and-shop/layout"

function SellPage() {
  var [image, updateImage] = useState({ key: null, signedUrl: null })

  async function getImage() {
    if (!image.key) {
      return
    }
    var signedUrl = await Storage.get(image.key)
    updateImage({ ...image, signedUrl })
  }

  async function onChange(e) {
    var file = e.target.files[0]
    if (!file) {
      return
    }
    var imageInfo = await Storage.put(file.name, file)
    updateImage({ ...image, key: imageInfo.key })
    console.log("image uploaded...", imageInfo)
  }

  return (
    <BuyLayout>
      <h1>This is the Sell Page!</h1>
      <input type="file" accept="image/png" onChange={e => onChange(e)} />
      <button onClick={getImage}>Get Image</button>

      {image.signedUrl && <img src={image.signedUrl} style={{ width: 500 }} />}
    </BuyLayout>
  )
}

export default withAuthenticator(SellPage, {
  includeGreetings: true,
})
```

3. Start the Mock Services

```javascript
amplify mock
```

*NOTE:* If we are using authentication, we will have to set it up globally to be able to use it locally.