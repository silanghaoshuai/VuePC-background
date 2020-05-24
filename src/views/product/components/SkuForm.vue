<template>
  <el-form label-width="120px">
    <el-form-item label="SPU 名称">
      <span>atguigu</span>
    </el-form-item>

    <el-form-item label="SKU 名称">
      <el-input placeholder="SKU 名称" v-model="skuInfo.skuName"></el-input>
    </el-form-item>

    <el-form-item label="价格">
      <el-input type="number" placeholder="SKU 价格" v-model="skuInfo.price"></el-input>
    </el-form-item>

    <el-form-item label="重量(千克)">
      <el-input type="number" placeholder="SKU 重量" v-model="skuInfo.weight"></el-input>
    </el-form-item>

    <el-form-item label="规格描述">
      <el-input type="textarea" placeholder="SKU 规格描述" rows="4" v-model="skuInfo.skuDesc"></el-input>
    </el-form-item>

    <el-form-item label="平台属性">
      <el-form inline label-width="80px">
        <el-form-item :label="attr.attrName" style="margin: 5px" v-for="attr in attrList" :key="attr.id">
          <el-select v-model="attr.attrId_valueId">
            <el-option :label="value.valueName" :value="attr.id + '_' +value.id"
              v-for="value in attr.attrValueList" :key="attr.id"></el-option>
          </el-select>
        </el-form-item>
      </el-form>
    </el-form-item>

    <el-form-item label="销售属性">
      <el-form inline label-width="80px">
        <el-form-item :label="attr.saleAttrName" style="margin: 5px" v-for="attr in spuSaleAttrList" :key="attr.id">
          <el-select v-model="attr.saleAttrValueId">
            <el-option :label="value" :value="value.id" v-for="value in attr.spuSaleAttrValueList" :key="value.id"></el-option>
          </el-select>
        </el-form-item>

      </el-form>
    </el-form-item>

    <el-form-item label="图片列表">
      <el-table border :data="spuImageList" @selection-change="handleSelectionChange">
        <el-table-column
          type="selection"
          width="55">
        </el-table-column>
        <el-table-column label="图片">
          <template slot-scope="{row}">
            <img :src="row.imgUrl" alt="" style="width: 150px;height:150px">
          </template>
        </el-table-column>
        <el-table-column label="名称" prop="imgName"></el-table-column>
        <el-table-column label="操作">
          <template slot-scope="{row}">
            <el-tag type="success" v-if="row.isDefault==='1'">默认</el-tag>
            <el-button type="primary" v-else @click="setDefault(row)">设为默认</el-button>
          </template>
        </el-table-column>
      </el-table>
    </el-form-item>

    <el-form-item>
      <el-button type="primary" @click="save">保存</el-button>
      <el-button @click="back">返回</el-button>
    </el-form-item>


  </el-form>
</template>

<script>
export default {
  name: 'SkuForm',

  data () {
    return {
      skuInfo: {
        category3Id: '',
        spuId: '',
        tmId: '',
        skuName: '',
        skuDesc: '',
        price: '',
        weight: '',
        skuDefaultImg: '',
        skuAttrValueList: [],
        skuSaleAttrValueList: [],
        skuImageList: [],
      },
      spu: {}, // 所属的SPU对象
      attrList: [], // 平台属性列表
      spuImageList: [], // SPU的图片列表
      spuSaleAttrList: [], // SPU的销售属性列表
      selectedImageList: [], // 所有选中图片对象的列表
    }
  },
  methods: {
    async save () {
      const {skuInfo, attrList, spuSaleAttrList, selectedImageList} = this
      skuInfo.skuAttrValueList = attrList.reduce((pre, attr) => {
        const attrId_valueId = attr.attrId_valueId
        if (attrId_valueId) {
          const [attrId, valueId] = attrId_valueId.split('_')
          pre.push({
            attrId,
            valueId
          })
        }

        return pre
      }, [])
      skuInfo.skuSaleAttrValueList = spuSaleAttrList.reduce((pre, attr) => {
        const saleAttrValueId = attr.saleAttrValueId
        if (saleAttrValueId) {
          pre.push({
            saleAttrValueId
          })
        }
        return pre
      }, [])
      skuInfo.skuImageList = selectedImageList.map(item => ({
        imgName: item.imgName,
        imgUrl: item.imgUrl,
        spuImgId: item.id,
        isDefault: item.isDefault
      }))
      // 发送添加SKU的请求
      const result = await this.$API.sku.addUpdate(skuInfo)
      if (result.code===200) {
        this.$message.success('保存SKU成功')
        // 重置数据
        this.resetData()
        // 通知父组件保存成功
        this.$emit('saveSuccess')
      } else {
        this.$message.error('保存SKU失败')
      }
    },

    back () {
      this.resetData()
      this.$emit('cancel')
    },
    resetData () {
      this.skuInfo = {
        category3Id: '',
        spuId: '',
        tmId: '',
        skuName: '',
        skuDesc: '',
        price: '',
        weight: '',
        skuDefaultImg: '',
        skuAttrValueList: [],
        skuSaleAttrValueList: [],
        skuImageList: [],
      },
      this.spu = {},
      this.attrList = []
      this.spuImageList = []
      this.spuSaleAttrList = []
      this.selectedImageList = []
    },
    handleSelectionChange (selection) {
      this.selectedImageList = selection
    },
    setDefault (spuImage) {
      this.spuImageList.forEach(item => item.isDefault = '0')

      spuImage.isDefault = '1'
      this.skuInfo.skuDefaultImg = spuImage.imgUrl
    },
    initAddData (spu) {
      this.spu = spu
      this.skuInfo.category3Id = spu.category3Id,
      this.skuInfo.spuId = spu.id
      this.skuInfo.tmId = spu.tmId
      this.getSpuInitData()
    },
    async getSpuInitData () {
      const {category1Id, category2Id, category3Id, id} = this.spu
      const promise1 = this.$API.attr.getList(category1Id, category2Id, category3Id)
      const promise2 = this.$API.sku.getSpuImageList(id)
      const promise3 = this.$API.sku.getSpuSaleAttrList(id)
      const results = await Promise.all([promise1, promise2, promise3])
      this.attrList = results[0].data
      const spuImageList = results[1].data
      spuImageList.forEach(item => item.isDefault = '0')
      this.spuImageList = spuImageList
      this.spuSaleAttrList = results[2].data
    }
  }
}
</script>

<style lang="less" scoped>

</style>
