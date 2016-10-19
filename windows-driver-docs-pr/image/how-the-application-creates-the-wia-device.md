---
title: How the Application Creates the WIA Device
author: windows-driver-content
description: How the Application Creates the WIA Device
MS-HAID:
- 'WIA\_drv\_basic\_2d834d2b-0f93-43f5-abec-07bea02d9952.xml'
- 'image.how\_the\_application\_creates\_the\_wia\_device'
MSHAttr:
- 'PreferredSiteName:MSDN'
- 'PreferredLib:/library/windows/hardware'
ms.assetid: f4268c61-11e5-4796-b7cb-80c8112be4d8
---

# How the Application Creates the WIA Device


## <a href="" id="ddk-how-the-application-creates-the-wia-device-si"></a>


When an application intends to use a WIA device driver, it calls the **IWiaDevMgr::CreateDevice** method (described in the Microsoft Windows SDK documentation). The WIA service first calls [**IStiUSD::LockDevice**](https://msdn.microsoft.com/library/windows/hardware/ff543829) to lock the WIA driver for mutually exclusive access. Next, the WIA service calls [**IWiaMiniDrv::drvInitializeWia**](https://msdn.microsoft.com/library/windows/hardware/ff544986) to create the initial WIA item tree structure. Finally, the WIA service unlocks the device driver by calling [**IStiUSD::UnLockDevice**](https://msdn.microsoft.com/library/windows/hardware/ff543843).

The **IWiaMiniDrv::drvInitializeWia** method should perform the following tasks.

1.  Cache the interface that the *pStiDevice* parameter points to for proper device locking. (For more information, see [**IWiaMiniDrv::drvLockWiaDevice**](https://msdn.microsoft.com/library/windows/hardware/ff544995).)

2.  Create the initial WIA item tree structure.

3.  Increment the current application connection count. This count is used to inform the driver whether an application is still connected. It also helps determine the proper action to take in [**IWiaMiniDrv::drvUnInitializeWia**](https://msdn.microsoft.com/library/windows/hardware/ff545010).

WIA items should be named with some logical meaning. Microsoft requires the following item names for Windows XP and later.

<a href="" id="root"></a>**Root**  
This is the term for the root item of the WIA item tree.

<a href="" id="flatbed"></a>**Flatbed**  
This is the term for a scanner that supports a flatbed scanner only, or a scanner that supports a flatbed scanner with a document feeder.

<a href="" id="feeder"></a>**Feeder**  
This is the term for a scanner that supports only a feeder.

The WIA service calls the **IWiaMiniDrv::drvInitializeWia** method in response to a WIA application's call to **IWiaDevMgr::CreateDevice** (described in the Windows SDK documentation). A consequence of this is that the WIA service calls the **IWiaMiniDrv::drvInitializeWia** method for each new client connection.

The **IWiaMiniDrv::drvInitializeWia** method should initialize any private structures and create the driver item tree. The driver item tree shows the layout of all WIA items supported by this WIA device. This method is used to create the initial tree structure only, *not* the contents (WIA properties). The WIA service will individually populate the WIA properties for the WIA driver items by making multiple calls to the [**IWiaMiniDrv::drvInitItemProperties**](https://msdn.microsoft.com/library/windows/hardware/ff544989) method.

All WIA devices have a root item, which is the parent to all WIA device items. To create a WIA device item the WIA driver should call the WIA service helper function, [**wiasCreateDrvItem**](https://msdn.microsoft.com/library/windows/hardware/ff549160).

The following example shows how to create a WIA device root item.

```
LONG lItemFlags = WiaItemTypeFolder|WiaItemTypeDevice|WiaItemTypeRoot;
IWiaDrvItem  *pIWiaDrvRootItem  = NULL;
HRESULT hr = 
    wiasCreateDrvItem(
                       lItemFlags, // item flags
                       bstrRootItemName, // item name ("Root")
                       bstrRootFullItemName, // item full name ("0000\Root")
                      (IWiaMiniDrv *)this, // this WIA driver object
                       sizeof(MINIDRIVERITEMCONTEXT), // size of context
                       NULL, // context
                       &amp;pIWiaDrvRootItem // created ROOT item
                      );                 // (IWiaDrvItem interface)

if(S_OK == hr){

  //
  // ROOT item was created successfully
  //

 }
```

To create a WIA child item, located directly under the root item created in the previous example, use code similar to the following.

**Note**  **** Notice that the [**IWiaDrvItem::AddItemToFolder**](https://msdn.microsoft.com/library/windows/hardware/ff543856) method is called to add the newly created child item to the root item.

 

```
LONG lItemFlags = WiaItemTypeFile|WiaItemTypeDevice|WiaItemTypeImage;
PMINIDRIVERITEMCONTEXT pItemContext    = NULL;
IWiaDrvItem           *pIWiaDrvNewItem = NULL;
HRESULT hr = 
    wiasCreateDrvItem(
                       lItemFlags, // item flags
                       bstrItemName,  // item name ("Flatbed")
                       bstrFullItemName,  // item full name ("0000\Root\Flatbed")
                      (IWiaMiniDrv *)this,  // this WIA driver object
     sizeof(MINIDRIVERITEMCONTEXT), // size of context
                      (PBYTE)&amp;pItemContext, // context
                      &amp;pIWiaDrvNewItem // created child item
                     );                // (IWiaDrvItem interface)  

if(S_OK == hr){

  //
  // A New WIA driver item was created successfully
  //

  hr = pIWiaDrvNewItem->AddItemToFolder(pIWiaDrvRootItem); // add the new item to the ROOT
  if(S_OK == hr){

     //
     // successfully created and added a new WIA driver item to 
     // the WIA driver item tree.
    //

   }
   pNewItem->Release();
   pNewItem = NULL;
 }
```

The following example shows an implementation of the **IWiaMiniDrv::drvInitializeWia** method.

```
HRESULT _stdcall CWIADevice::drvInitializeWia(
  BYTE        *pWiasContext,
  LONG        lFlags,
  BSTR        bstrDeviceID,
  BSTR        bstrRootFullItemName,
  IUnknown    *pStiDevice,
  IUnknown    *pIUnknownOuter,
  IWiaDrvItem **ppIDrvItemRoot,
  IUnknown    **ppIUnknownInner,
  LONG        *plDevErrVal)
{
  //
  // If the caller did not pass in the correct parameters,
 // then fail the call with E_INVALIDARG.
  //

  if (!pWiasContext) {
      return E_INVALIDARG;
  }

  if (!plDevErrVal) {
      return E_INVALIDARG;
  }

  HRESULT hr = S_OK;

  *plDevErrVal = 0;
  *ppIDrvItemRoot = NULL;
  *ppIUnknownInner = NULL;

  if (m_pStiDevice == NULL) {

      //
      // save STI device interface for locking
      //

      m_pStiDevice = (IStiDevice *)pStiDevice;
  }

  //
  // build WIA item tree
  //

  LONG lItemFlags = WiaItemTypeFolder|WiaItemTypeDevice|WiaItemTypeRoot;

  IWiaDrvItem  *pIWiaDrvRootItem  = NULL;

  //
  // create the ROOT item of the WIA device.  This name should NOT be 
  // localized in different languages. "Root" is used by WIA drivers.
  //

  BSTR bstrRootItemName = SysAllocString(WIA_DEVICE_ROOT_NAME);
  if(!bstrRootItemName) {
      return E_OUTOFMEMORY;
  }

  hr = wiasCreateDrvItem(lItemFlags,  // item flags
               bstrRootItemName,  // item name ("Root")
               bstrRootFullItemName,  // item full name ("0000\Root")
                (IWiaMiniDrv *)this,  // this WIA driver object
      sizeof(MINIDRIVERITEMCONTEXT),  // size of context
                               NULL,  // context
                 &amp;pIWiaDrvRootItem);  // created ROOT item
                                      // (IWiaDrvItem interface)
  if (S_OK == hr) {

    //
    // ROOT item was created successfully, save the newly created Root
    // item in the pointer given by the WIA service (ppIDrvItemRoot).
    //

      *ppIDrvItemRoot = pIWiaDrvRootItem;

    //
    // Create a child item  directly under the Root WIA item
    //

      lItemFlags = WiaItemTypeFile|WiaItemTypeDevice|WiaItemTypeImage;

      PMINIDRIVERITEMCONTEXT pItemContext    = NULL;
      IWiaDrvItem           *pIWiaDrvNewItem = NULL;

      //
      // create a name for the WIA child item.  "Flatbed" is used by 
      // WIA drivers that support a flatbed scanner.
      //

      BSTR bstrItemName = SysAllocString(WIA_DEVICE_FLATBED_NAME);

      if (bstrItemName) {

          WCHAR  wszFullItemName[MAX_PATH + 1] = {0};
          _snwprintf(wszFullItemName,(sizeof(wszFullItemName) / sizeof(WCHAR)) - 1,L"%ls\\%ls",
                   bstrRootFullItemName,bstrItemName);

        BSTR bstrFullItemName = SysAllocString(wszFullItemName);
        if (bstrFullItemName) {
          hr = wiasCreateDrvItem(lItemFlags,  // item flags
                               bstrItemName,  // item name ("Flatbed")
                             trFullItemName,  // item full name ("0000\Root\Flatbed")
                        (IWiaMiniDrv *)this,  // this WIA driver object
               sizeof(MINIDRIVERITEMCONTEXT), // size of context
                       (BYTE**)&amp;pItemContext, // context
                        &amp;pIWiaDrvNewItem);    // created child item
                                              // (IWiaDrvItem interface)

            if (S_OK == hr) {

                //
                // A New WIA driver item was created successfully
                //

                hr = pIWiaDrvNewItem->AddItemToFolder(pIWiaDrvRootItem); // add the new item to the ROOT
  if (S_OK == hr) {

                    //
                    // successfully created and added a new WIA 
                    // driver item to the WIA driver item tree.
                    //

                }

                //
                // The new item is no longer needed, because it has
                // been passed to the WIA service.
                //

                pIWiaDrvNewItem->Release();
                pIWiaDrvNewItem = NULL;
            }
            SysFreeString(bstrFullItemName);
            bstrFullItemName = NULL;
        } else {
            hr = E_OUTOFMEMORY;
        }
        SysFreeString(bstrItemName);
        bstrItemName = NULL;
    } else {
        hr = E_OUTOFMEMORY;
    }
  }

  //
  // increment application connection count
  //

  if(S_OK == hr){
    InterlockedIncrement(&amp;m_lClientsConnected);
  }

  return hr;
}
```

 

 


--------------------
[Send comments about this topic to Microsoft](mailto:wsddocfb@microsoft.com?subject=Documentation%20feedback%20%5Bimage\image%5D:%20How%20the%20Application%20Creates%20the%20WIA%20Device%20%20RELEASE:%20%288/17/2016%29&body=%0A%0APRIVACY%20STATEMENT%0A%0AWe%20use%20your%20feedback%20to%20improve%20the%20documentation.%20We%20don't%20use%20your%20email%20address%20for%20any%20other%20purpose,%20and%20we'll%20remove%20your%20email%20address%20from%20our%20system%20after%20the%20issue%20that%20you're%20reporting%20is%20fixed.%20While%20we're%20working%20to%20fix%20this%20issue,%20we%20might%20send%20you%20an%20email%20message%20to%20ask%20for%20more%20info.%20Later,%20we%20might%20also%20send%20you%20an%20email%20message%20to%20let%20you%20know%20that%20we've%20addressed%20your%20feedback.%0A%0AFor%20more%20info%20about%20Microsoft's%20privacy%20policy,%20see%20http://privacy.microsoft.com/default.aspx. "Send comments about this topic to Microsoft")

