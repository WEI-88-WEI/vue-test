<template>
  <div>
    <el-form ref="form" :model="form" :rules="formRules" inline>
      <el-form-item label="平台 / 站点 / 店铺" prop="associateList">
        <CascadeSelector
          v-model="form.associateList"
          api-key="platformSiteShop"
          not-required-params
          placeholder="请选择平台 / 站点 / 店铺"
          :props="{ multiple: true }"
          collapse-tags
          @change="
            form.styleIdList = []
          "
        />
      </el-form-item>
      <el-form-item label="BU" prop="buUser">
        <CascadeSelector
          v-model="form.buUser"
          :props="{ multiple: true }"
          collapse-tags
          api-key="BUTree"
          :permission="false"
          placeholder="BU"
          @change="
            form.styleIdList = []
          "
        />
      </el-form-item>
      <el-form-item label="CT" prop="combatTeam">
        <ErpSelection
          v-model="form.combatTeam "
          placeholder="CT"
          :select-options="ctList"
        />
      </el-form-item>
      <el-form-item label="Style" prop="styleIdList">
        <virtualized-select
          v-model="form.styleIdList"
          :options="styleList"
          :props="{
            label: 'style',
            value: 'styleId',
          }"
          collapse-tags
          multiple
          placeholder="请选择Style"
        />
      </el-form-item>
      <el-form-item label="类目" prop="categoryIdList">
        <CascadeSelector
          v-model="form.categoryIdList"
          collapse-tags
          :permission="false"
          api-key="categoryTreeRefactor"
          :props="{emitPath:false,multiple:true}"
          placeholder="请选择类目"
        />
      </el-form-item>
      <el-form-item label="品牌" prop="brandIdList">
        <ErpSelection
          v-model="form.brandIdList"
          multiple
          placeholder="请选择品牌"
          api-key="brandList"
        />
      </el-form-item>
      <el-form-item label="销售季节" prop="saleSeasonList">
        <ErpSelection
          v-model="form.saleSeasonList"
          multiple
          placeholder="请选择销售季节"
          api-key="season"
        />
      </el-form-item>
      <el-form-item label="Style定位" prop="positionList">
        <ErpSelection
          v-model="form.positionList"
          placeholder="请选择Style定位"
          api-key="stylePosition"
          multiple
        />
      </el-form-item>
      <el-form-item label="测算成本参考月份" prop="dateMonthList">
        <vxe-input
          v-model="form.dateMonthList"
          type="month"
          clearable
          multiple
          placeholder="请选择一个或多个月"
          value-format="yyyy-MM"
          label-format="yyyy-MM"
          :disabled-method="disabledDate"
        />
      </el-form-item>
      <div style="margin: 0px 0px 18px 0px; display: inline-block">
        <el-button type="primary" :loading="searchLoading" @click="handleQuery()">查询</el-button>
        <el-button @click="handleReset()">重置</el-button>
      </div>
    </el-form>
    <vxe-toolbar custom>
      <template #buttons>
        <el-button v-permission="'profitabilityMeasurement:import'" type="primary" @click="importVisible=true">导入调整</el-button>
        <el-button v-permission="'profitabilityMeasurement:vatTaxRate'" type="primary" @click="vatTaxRateVisible=true">VAT税率配置</el-button>
        <el-button v-permission="'profitabilityMeasurement:commissionRebateRatio'" type="primary" @click="commissionRateVisible=true">佣金比例配置</el-button>
        <el-button v-permission="'profitabilityMeasurement:gbTaxRule'" type="primary" @click="gbTaxRuleVisible=true">GB税费规则配置</el-button>
        <el-button type="primary" @click="handleBatchEdit">批量编辑</el-button>
        <el-button v-permission="'profitabilityMeasurement:export'" @click="handleExport">导出</el-button>
        <span v-if="noGbVatConfigNum" style="color: #E6A23C;margin-left: 10px">{{ `GB站点存在${noGbVatConfigNum}个Style待维护其是否计算VAT税费，请注意及时维护，否则无法完成利润测算` }}</span>
      </template>
    </vxe-toolbar>
    <vxe-table
      id="profitability-measurement"
      ref="table"
      :loading="searchLoading"
      :max-height="maxHeight"
      align="center"
      :custom-config="{storage: true,mode: 'popup'}"
      :data="tableData"
      show-overflow
      :scroll-x="{ enabled: true }"
      :scroll-y="{ enabled: true }"
      :row-config="{ isHover: true }"
      keep-source
      :edit-rules="validRules"
      :edit-config="{trigger: 'click', mode: 'cell',showStatus: true}"
      :column-config="{ resizable: true,maxFixedSize:20 }"
      class="xTable"
      @checkbox-all="selectChangeEvent"
      @checkbox-change="selectChangeEvent"
    >
      <vxe-column type="checkbox" width="40" fixed="left" />
      <vxe-table-column field="associateName" title="平台站点店铺" min-width="120" fixed="left">
        <template #default="{ row }">
          <span v-if="row.associateName==='total'">{{ 'Total-计算' }}</span>
          <span v-else>{{ row.associateName }}</span>
          <el-tooltip v-if="row.associateName==='total'" effect="dark" placement="top">
            <div slot="content">
              <div>
                此初始数据结果基于搜索结果汇总，用于支持独立且自定义计算汇总结果的利润数据。
              </div>
              <div>
                注：考虑系统性能，其它行的编辑后计算结果无法与此行相互影响
              </div>
            </div>
            <i class="el-icon-question" style="margin-left: 5px" />
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="styleName" title="Style" min-width="100" fixed="left" />
      <vxe-table-column field="categoryName" title="类目" min-width="120" fixed="left" />
      <vxe-table-column field="brandName" title="品牌" min-width="80" fixed="left" />
      <vxe-table-column field="buName" title="BU" min-width="80" fixed="left" />
      <vxe-table-column field="combatTeamCode" title="CT" min-width="80" fixed="left" />
      <vxe-table-column field="saleSeason" title="销售季节" min-width="80" fixed="left" />
      <vxe-table-column field="position" title="Style定位" min-width="80" fixed="left" />
      <vxe-table-column field="price" title="售价$" min-width="100" :edit-render="{}">
        <template #edit="{ row }">
          <vxe-input
            v-model="row.price"
            align="center"
            placeholder="请输入"
            min="0"
            type="float"
            :controls="false"
            step="0.01"
            clearable
            @input="handlePrice(row,1)"
          />
        </template>
      </vxe-table-column>
      <template #header>
        <el-tooltip effect="dark" content="含税价" placement="top">
          <span>售价$</span>
        </el-tooltip>
      </template>
      <vxe-table-column field="localPrice" title="售价(本地货币)" min-width="140" :edit-render="{}">
        <template #valid="{row,content}">
          <span v-if="row.associateName==='total'" style="color: #F56C6C">{{ '无需输入' }}</span>
          <span v-else style="color: #F56C6C">{{ content }}</span>
        </template>
        <template #edit="{ row }">
          <span v-if="row.associateName==='total'">{{ row.localPrice }}</span>
          <vxe-input
            v-else
            v-model="row.localPrice"
            align="center"
            placeholder="请输入"
            min="0"
            type="float"
            step="0.01"
            clearable
            :controls="false"
            @input="handlePrice(row,2)"
          />
        </template>
        <template #header>
          <el-tooltip effect="dark" content="含税价" placement="top">
            <span>售价(本地货币)</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="calCostMonth" title="测算成本参考月份" min-width="120" :edit-render="{}">
        <template #valid="{row,content}">
          <span v-if="row.associateName==='total'" style="color: #F56C6C">{{ '无需输入' }}</span>
          <span v-else style="color: #F56C6C">{{ content }}</span>
        </template>
        <template #edit="{ row }">
          <span v-if="row.associateName==='total'">{{ row.calCostMonth }}</span>
          <vxe-input
            v-else
            v-model="row.calCostMonth"
            align="center"
            placeholder="请选择"
            type="month"
            clearable
            multiple
            transfer
            value-format="yyyy-MM"
            label-format="yyyy-MM"
            :disabled-method="disabledDate"
          />
        </template>
      </vxe-table-column>
      <vxe-table-column field="estimatedSold" title="预计销量" min-width="120" :edit-render="{}">
        <template #edit="{ row }">
          <vxe-input
            v-model="row.estimatedSold"
            align="center"
            placeholder="请输入"
            min="0"
            type="integer"
            :controls="false"
            step="1"
            clearable
          />
        </template>
      </vxe-table-column>
      <vxe-table-column field="advSaleRate" title="广销比" :formatter="formatter" min-width="100" :edit-render="{}">
        <template #edit="{ row }">
          <vxe-input
            v-model="row.advSaleRate"
            align="center"
            placeholder="请输入"
            min="-100"
            max="0"
            type="float"
            :controls="false"
            :digits="4"
            step="0.0001"
            clearable
          />
        </template>
        <template #header>
          <el-tooltip
            effect="dark"
            content="历史广告费/收入income；广告费用：advertiseFeeSP + advertiseFeeSB + advertiseFeeSBVideo + advertiseFeeSD + advertiseFeeElse"
            placement="top"
          >
            <span>广销比</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="returnRate" title="退货率" :formatter="formatter" min-width="100" :edit-render="{}">
        <template #header>
          <el-tooltip effect="dark" content="所选月份退货数量合计/所选月份销量合计" placement="top">
            <span>退货率</span>
          </el-tooltip>
        </template>
        <template #edit="{ row }">
          <vxe-input
            v-model="row.returnRate"
            align="center"
            placeholder="请输入"
            min="0"
            max="100"
            type="float"
            :controls="false"
            :digits="4"
            step="0.0001"
            clearable
          />
        </template>
      </vxe-table-column>
      <vxe-table-column field="historySold" title="历史销量" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="所选月份历史销量合计" placement="top">
            <span>历史销量</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="historyReturn" title="历史退货量" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="所选月份历史退货量合计" placement="top">
            <span>历史退货量</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="storageFeeRate" :formatter="formatter" title="仓储费%" min-width="110" :edit-render="{}">
        <template #header>
          <el-tooltip effect="dark" content="历史月度仓储及操作费/收入income；月度仓储及长仓费：storageFee + longStorageFee" placement="top">
            <span>仓储费%</span>
          </el-tooltip>
        </template>
        <template #edit="{ row }">
          <vxe-input
            v-model="row.storageFeeRate"
            align="center"
            placeholder="请输入"
            min="-100"
            max="0"
            type="float"
            :controls="false"
            :digits="4"
            step="0.0001"
            clearable
          />
        </template>
      </vxe-table-column>
      <vxe-table-column field="otherFeeRate" :formatter="formatter" title="其它费用%" min-width="120" :edit-render="{}">
        <template #header>
          <el-tooltip
            effect="dark"
            content="历史其他费用/收入income；其他费用：liquidationCost + liquidationFirstPass + liquidationTax + liquidationAmount + liquidationFee + DealFee + foreignPromotionFee + removalFee + ServiceFee"
            placement="top"
          >
            <span>其它费用%</span>
          </el-tooltip>
        </template>
        <template #edit="{ row }">
          <vxe-input
            v-model="row.otherFeeRate"
            align="center"
            placeholder="请输入"
            min="-100"
            max="0"
            type="float"
            :controls="false"
            :digits="4"
            step="0.0001"
            clearable
          />
        </template>
      </vxe-table-column>
      <vxe-table-column field="purchaseCost" title="采购成本" min-width="120" :edit-render="{}">
        <template #header>
          <el-tooltip effect="dark" content="默认取值：所选月份的平均单个产品的采购成本*预计销量" placement="top">
            <span>采购成本</span>
          </el-tooltip>
        </template>
        <template #edit="{ row }">
          <vxe-input
            v-model="row.purchaseCost"
            align="center"
            placeholder="请输入"
            max="0"
            type="float"
            :controls="false"
            step="0.01"
            clearable
          />
        </template>
      </vxe-table-column>
      <vxe-table-column field="firstPassFee" title="头程费用" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="默认取值：所选月份的平均单个产品的头程费用*预计销量" placement="top">
            <span>头程费用</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="taxFee" title="关税费用" min-width="120" :edit-render="{}">
        <template #header>
          <el-tooltip effect="dark" content="默认取值：所选月份的平均单个产品的关税费用*预计销量" placement="top">
            <span>关税费用</span>
          </el-tooltip>
        </template>
        <template #edit="{ row }">
          <vxe-input
            v-model="row.taxFee"
            align="center"
            placeholder="请输入"
            max="0"
            type="float"
            :controls="false"
            step="0.01"
            clearable
          />
        </template>
      </vxe-table-column>
      <vxe-table-column field="finalDeliveryFee" title="尾程配送费用" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="所选月份的平均单个产品的尾程配送费用*预计销量" placement="top">
            <span>尾程配送费用</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="platformCommission" title="平台佣金" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="售价* 佣金比例 * -1*预计销量" placement="top">
            <span>平台佣金</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="advFee" title="广告费用" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="广销比*售价$*预计销量" placement="top">
            <span>广告费用</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="storageFee" title="仓储费" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="仓储费%*售价$*预计销量" placement="top">
            <span>仓储费</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="vatCommissionFee" title="VAT税费" min-width="110">
        <template #header>
          <el-tooltip effect="dark" content="VAT税费=(售价$/(1+VAT税率))*VAT税率*-1*预计销量" placement="top">
            <span>VAT税费</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="otherFee" title="其他费用" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="其它费用%*售价$*预计销量" placement="top">
            <span>其他费用</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="liquidationFee" title="Liquidation费用" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="所选月份的liquidation合计/所选月份的income合计*售价$*预计销量" placement="top">
            <span>Liquidation费用</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="managementFee" title="管理费用" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="售价$*7%*-1*预计销量" placement="top">
            <span>管理费用</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="returnAmount" title="退货金额" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="退货率*售价$*预计销量*-1" placement="top">
            <span>退货金额</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="returnDeliveryFee" title="退货配送费" min-width="100">
        <template #header>
          <el-tooltip
            effect="dark"
            content="退货费用*退货率*预计销量
            amazon: 退货费用=所选月份的平均单个产品的尾程配送费用/2
            ebay:退货费用=0
            shein: 退货费用=0.65$
            walmart:退货费用=所选月份的平均单个产品的尾程配送费用"
            placement="top"
          >
            <span>退货配送费</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="purchaseCostBack" title="采购成本冲回" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="退货率*采购成本*-1" placement="top">
            <span>采购成本冲回</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="firstDutyBack" title="头程关税冲回" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="退货率*（头程费用+关税费用）*-1" placement="top">
            <span>头程关税冲回</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="commissionBack" title="佣金冲回" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="退货率*预计销量*售价$*佣金冲回比例" placement="top">
            <span>佣金冲回</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="vatBack" title="VAT冲回" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="退货率*预计销量*VAT税费VAT税费=(售价$/(1+VAT税率))*VAT税率" placement="top">
            <span>VAT冲回</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="grossProfitAmount" title="产品毛利金额(不含货损赔偿)" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="售价$*预计销量+采购成本+头程费用+关税费用+尾程配送费用+平台佣金+广告费用+仓储费+VAT税费+其它费用+退货金额+退货配送费+采购成本冲回+头程关税冲回+佣金冲回+VAT冲回" placement="top">
            <span>产品毛利金额(不含货损赔偿)</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="grossProfitRate" :formatter="formatter" title="产品毛利率" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="产品毛利金额(不含货损赔偿）/(售价$*预计销量)" placement="top">
            <span>产品毛利率</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="netProfitAmount" title="产品净利金额-不含货损赔偿" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="产品毛利金额(不含货损赔偿）+管理费用" placement="top">
            <span>产品净利金额-不含货损赔偿</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="netProfitRate" :formatter="formatter" title="产品净利率" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="产品净利金额(不含货损赔偿）/(售价$*预计销量)" placement="top">
            <span>产品净利率</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="domesticPrecipitatedCashFlow" title="沉淀现金流（国内发货）" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="产品毛利金额(不含货损赔偿）-采购成本-采购成本冲回-liquidation费用" placement="top">
            <span>沉淀现金流（国内发货）</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="aboardPrecipitatedCashFlow" title="沉淀现金流（海外发货）-用于EOL产品测算现金流" min-width="100">
        <template #header>
          <el-tooltip effect="dark" content="产品毛利金额(不含货损赔偿）-采购成本-采购成本冲回-头程费用-关税费用-头程关税冲回-liquidation费用" placement="top">
            <span>沉淀现金流（海外发货）-用于EOL产品测算现金流</span>
          </el-tooltip>
        </template>
      </vxe-table-column>
      <vxe-table-column field="operate" title="操作" width="100" fixed="right">
        <template #default="{ row,rowIndex }">
          <el-button type="primary" :loading="row.loading" size="mini" @click="handleEdit(row,rowIndex)">完成</el-button>
        </template>
      </vxe-table-column>
    </vxe-table>

    <Paging ref="pager" :pager="pager" :config="pagerConfig" end @update="pagerUpdate" />

    <DialogImport :import-visible.sync="importVisible" @refresh="handleQuery" />
    <DialogVatTaxRate :vat-tax-rate-visible.sync="vatTaxRateVisible" @refresh="handleQuery" />
    <DialogCommissionRate :commission-rate-visible.sync="commissionRateVisible" @refresh="handleQuery" />
    <DialogGbTaxRule :gb-tax-rule-visible.sync="gbTaxRuleVisible" @refresh="handleQuery" />
    <DialogEdit :edit-visible.sync="editVisible" :select-list="selectList" :save-form="saveForm" @editDone="editDone" />
  </div>
</template>

<script>
import ErpSelection from '@/components/ErpSelection'
import VirtualizedSelect from '@/components/VirtualizedSelect'
import CascadeSelector from '@/components/CascadeSelector'
import Paging from '@/components/Paging'
import { debounceGetTableMaxHeight, getCommodityDictByKey } from '@/utils'
import { getStyleList } from '@/api/erp-select'
import { getUserDefaultSite } from '@/api/erp/common'
import { profitList, profitExport, profitCal,
  profitPriceConvert, noGbVatConfigNum,
} from '@/api/erp/profitability-measurement'
import DialogImport from './DialogImport.vue'
import DialogVatTaxRate from './DialogVatTaxRate.vue'
import DialogCommissionRate from './DialogCommissionRate.vue'
import DialogGbTaxRule from './DialogGbTaxRule.vue'
import DialogEdit from './DialogEdit.vue'
import XEUtils from 'xe-utils'
import { mapGetters } from 'vuex'

import dayjs from 'dayjs'
import 'dayjs/locale/zh-cn'
dayjs.locale('zh-cn')

export default {
  components: {
    ErpSelection, VirtualizedSelect, CascadeSelector,
    Paging,
    DialogImport, DialogVatTaxRate, DialogCommissionRate, DialogGbTaxRule, DialogEdit,
  },
  data() {
    return {
      form: {
        associateList: [],
        buUser: [],
        combatTeam: undefined,
        styleIdList: [],
        categoryIdList: [],
        saleSeasonList: [],
        positionList: [],
        brandIdList: [],
        dateMonthList: Array.from({ length: 12 }, (_, i) =>
          dayjs().subtract(i + 1, 'month').format('YYYY-MM')
        ).join(','),
      },
      formRules: {
        associateList: [
          { required: true, message: '请选择平台/站点/店铺', trigger: ['blur', 'change'] },
        ],
      },
      searchLoading: false,
      pager: { current: 1, size: 20, total: 0 },
      pagerConfig: ['current', 'size', 'total'],
      tableData: [],
      maxHeight: 500,
      xTableRef: null,
      styleList: [],
      selectList: [],
      saveForm: {},
      formChange: false,
      footerMergeCells: [],
      defaultPlatformSiteShop: [],
      noGbVatConfigNum: undefined,
      // GB站点
      GB: 3,
      timer: null,
      controller: null,

      exportLoading: false,

      importVisible: false,
      vatTaxRateVisible: false,
      commissionRateVisible: false,
      gbTaxRuleVisible: false,
      editVisible: false,
    }
  },
  computed: {
    ...mapGetters('dictionary', ['wmsAllCommodityDict']),
    ctList() {
      return getCommodityDictByKey({
        key: 'COMBAT_TEAM',
        dict: this.wmsAllCommodityDict,
      })
    },
    platformSiteShopParams() {
      const platformSiteShopMap = this.form.associateList.reduce((acc, cur) => {
        const [platformId, siteId, shopId] = cur
        platformId && acc.platformId.add(platformId)
        siteId && acc.siteId.add(siteId)
        shopId && acc.shopId.add(shopId)
        return acc
      }, { platformId: new Set(), siteId: new Set(), shopId: new Set() })
      return {
        platformId: [...platformSiteShopMap.platformId],
        siteId: [...platformSiteShopMap.siteId],
        shopId: [...platformSiteShopMap.shopId],
      }
    },
    buIdAndGroupId() {
      const _buId = new Set()
      const _groupId = new Set()
      if (this.form.buUser.length) {
        this.form.buUser.forEach(item => {
          item?.[0] && _buId.add(item[0])
          item?.[1] && _groupId.add(item[1])
        })
      }
      return { groupId: [..._groupId], buId: [..._buId] }
    },
    styleParams() {
      return {
        userId: this.$store.getters.userId,
        buId: this.buIdAndGroupId.buId, groupId: this.buIdAndGroupId.groupId,
        combatTeamList: this.form.combatTeam ? [this.form.combatTeam] : [],
        ...this.platformSiteShopParams,
      }
    },
    queryParams() {
      return {
        ...this.form, ...this.pager,
        userId: this.$store.getters.userId,
        buIdList: this.buIdAndGroupId.groupId,
        dateMonthList: this.form.dateMonthList.split(','),
      }
    },
    validRules() {
      const validFeild = [
        'price', 'localPrice', 'calCostMonth', 'estimatedSold',
        'advSaleRate', 'returnRate', 'storageFeeRate', 'otherFeeRate',
        'purchaseCost', 'taxFee',
      ]
      const validRulesLists = {}
      validFeild.forEach(item => {
        validRulesLists[item] = [{ required: true, message: '请输入' }]
      })
      return validRulesLists
    },
  },
  watch: {
    'styleParams': {
      handler() {
        this.getStyleList()
      },
    },
    form: {
      handler() {
        this.formChange = true
      },
      deep: true,
    },
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.debounceGetTableMaxHeight)
    this.timer && clearTimeout(this.timer)
  },
  mounted() {
    this.$nextTick(() => {
      this.xTableRef = this.$refs.table
    })

    this.getDefaultPlatformSiteShop()

    this.debounceGetTableMaxHeight = debounceGetTableMaxHeight.bind(this)
    this.debounceGetTableMaxHeight()
    window.addEventListener('resize', this.debounceGetTableMaxHeight)
  },
  methods: {
    editDone(arr) {
      arr.forEach(item => {
        this.$set(this.tableData, item.editIndex, item)
      })
    },
    handleBatchEdit() {
      if (this.selectList.length === 0) {
        this.$message.warning('请选择一条数据')
        return
      }
      // 如果站点不一样，不允许批量编辑
      if (new Set(this.selectList.filter(item => item.siteId && item.siteId !== -1).map(item => item.siteId)).size > 1) {
        this.$message.warning('站点不一致，不允许批量编辑')
        return
      }
      this.editVisible = true
    },
    async handleExport() {
      this.exportLoading = true
      const { success, msg } = await profitExport(this.saveForm).finally(() => {
        this.exportLoading = false
      })
      if (success) {
        this.$message({
          type: 'success',
          dangerouslyUseHTMLString: true,
          message: `${msg || '导出成功'} <a target="_blank" style="background-color: #67C23A;color: #FFFFFF;border-radius: 5px;padding: 5px 10px;display: inline-block" href="/data-download/download">去查看</a>`,
          duration: 5000,
        })
      } else {
        this.$message.error(msg || '导出失败')
      }
    },
    validateVat(row) {
      return new Promise((resolve, reject) => {
        if (this.saveForm.associateList.find(item => item?.[1] === this.GB) &&
        (row.vatCommissionFee === null || row.vatCommissionFee === undefined || row.vatCommissionFee === '')
        ) {
          this.$message.error('该Style未在“GB税费规则配置”页面维护好其是否计算VAT税费，无法完成利润测算')
          resolve(false)
        } else {
          resolve(true)
        }
      })
    },
    async handleMonthChange(row, rowIndex) {
      const res = await this.validateVat(row)
      if (!res) return
      if (!row.calCostMonth) return
      row.loading = true
      if (this.controller) {
        this.controller.abort()
        setTimeout(() => {
          row.loading = true
        })
      }
      this.controller = new AbortController()
      const signal = this.controller.signal
      const { datas: { records = [] }} = await profitList({
        associateList: [[row.platformId, row.siteId, row.shopId]],
        styleIdList: [row.styleId],
        dateMonthList: row.calCostMonth.split(','),
      }, signal).finally(() => {
        row.searchLoading = false
        this.controller = null
      })
      this.$set(this.tableData, rowIndex, records?.[0] || {})
      this.$message.success('修改成功')
    },
    /**
     * 处理价格转换的函数。
     * @param {number} type - 转换类型的标识符。
     *     - 1：将美元转换为本地货币。
     *     - 2：将本地货币转换为美元。
     */
    async handlePrice(row, type) {
      if (row.associateName === 'total') return
      this.timer && clearTimeout(this.timer)
      this.timer = setTimeout(async() => {
        const { siteId, price, localPrice } = row
        const priceParmas = type === 1 ? price : localPrice
        const field = type === 1 ? 'localPrice' : 'price'
        const { datas } = await profitPriceConvert({
          price: priceParmas,
          siteId,
          type,
        })
        XEUtils.set(row, field, datas)
      }, 500)
    },
    async handleEdit(row, rowIndex) {
      const res = await this.validateVat(row)
      if (!res) return
      const errMap = await this.xTableRef?.fullValidate(row).catch(errMap => errMap)
      if (errMap) {
        const msgList = []
        Object.entries(errMap).forEach(([key, errList]) => {
          errList.forEach(params => {
            const { rowIndex, column, rules, row } = params
            rules.forEach(rule => {
              if (row.associateName === 'total') {
                if (key !== 'calCostMonth' && key !== 'localPrice') {
                  msgList.push(`请将第 ${rowIndex + 1} 行 ${column.title} 信息填写完整`)
                }
              } else {
                msgList.push(`请将第 ${rowIndex + 1} 行 ${column.title} 信息填写完整`)
              }
            })
          })
        })
        if (msgList.length) {
          this.$message({
            type: 'error',
            dangerouslyUseHTMLString: true,
            message: msgList.join('<br>'),
          })
          return
        }
      }
      row.loading = true
      const { datas = {}} = await profitCal({
        profitQuery: this.saveForm,
        profitVO: row,
      }).finally(() => {
        row.loading = false
      })
      this.$set(this.tableData, rowIndex, datas)
      this.$message.success('修改成功')
    },
    async getDefaultPlatformSiteShop() {
      const { datas = [] } = await getUserDefaultSite({ userId: this.$store.getters.userId })
      this.defaultPlatformSiteShop = [datas]
      this.form.associateList = this.defaultPlatformSiteShop

      this.handleQuery()
    },
    async getStyleList() {
      const { data = [] } = await getStyleList(this.styleParams)
      this.styleList = data
    },
    async getTableData() {
      this.searchLoading = true
      const { datas: { records = [], pager = {}}, remark } = await profitList(this.queryParams).finally(() => {
        this.searchLoading = false
        this.formChange = false
      })
      remark && records.length && records.unshift({ ...remark })
      this.tableData = records.map((item, index) => {
        return {
          ...item,
          loading: false,
          editIndex: index,
        }
      })
      this.pager.total = pager.total ?? 0
    },
    async getNoGbVatConfigNum() {
      const { datas } = await noGbVatConfigNum()
      this.noGbVatConfigNum = datas
    },
    handleReset() {
      this.$refs?.form?.resetFields()
      this.form.associateList = this.defaultPlatformSiteShop
      this.noGbVatConfigNum = undefined
    },
    handleQuery() {
      this.$refs?.form.validate(async(valid) => {
        if (!valid) return
        this.saveForm = { ...this.queryParams }
        if (this.formChange) {
          this.pager.current = 1
        }
        await this.getTableData()
        this.noGbVatConfigNum = undefined
        if (this.saveForm.associateList.find(item => item?.[1] === this.GB)) {
          await this.getNoGbVatConfigNum()
        }
      })
    },
    pagerUpdate(val) {
      this.pager = val
      this.getTableData()
    },
    formatter({ cellValue }) {
      if (cellValue === null) {
        return ''
      } else {
        return `${(cellValue * 100).toFixed(2)} %`
      }
    },
    disabledDate({ date }) {
      return date.getTime() < +new Date('2023-01-01 00:00:00')
    },
    selectChangeEvent({ records }) {
      this.selectList = records
    },
  },
}
</script>

<style lang="scss" scoped>
::v-deep .el-dialog__body {
  padding: 5px 20px;
}
// ::v-deep .vxe-table--main-wrapper {
//   display: flex;
//   flex-direction: column;
// }
// ::v-deep .vxe-table--body-wrapper {
//   order: 1;

// }
// ::v-deep .vxe-table--footer-wrapper{
//   background-color: #f8f8f9;
//   margin-top: -1px !important;
//   &::-webkit-scrollbar {
//     display: none;
//   }
// }
// ::v-deep .xTable .vxe-table--empty-placeholder{
//   top: 70px !important;
// }
// ::v-deep .vxe-table--fixed-left-wrapper{
//   // 调整滚动条遮挡
//   height: calc(100% - 17px) !important;
//   & .vxe-table--body-wrapper{
//     top: 80px !important;
//     // 调整滚动条遮挡
//     max-height: calc(100% - 80px) !important;
//     &::-webkit-scrollbar {
//       display: none;
//     }
//   }
//   & .vxe-table--footer-wrapper{
//     top: 40px !important;
//   }
// }
// ::v-deep .vxe-table--fixed-right-wrapper{
//   // 调整滚动条遮挡
//   height: calc(100% - 17px) !important;
//   & .vxe-table--body-wrapper{
//     top: 80px !important;
//     // 调整滚动条遮挡
//     max-height: calc(100% - 63px) !important;
//   }
//   & .vxe-table--footer-wrapper{
//     top: 40px !important;
//   }
// }
</style>
